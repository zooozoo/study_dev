# ALTER TABLE 한 줄이 배포를 멈추는 이유로 배우는 MySQL 온라인 DDL

운영 중인 테이블에 컬럼을 하나 추가하는 일이 왜 어떤 때는 0.04초에 끝나고 어떤 때는 테이블 전체를 복사하는지를 정리한다. `ALGORITHM=INSTANT`, "테이블 rebuild", 그리고 마이그레이션 한 문장을 둘로 나눠야 하는 이유가 핵심이다.

작성일: 2026-09-11 · 실측 환경: MySQL 8.0.46 (로컬 Docker), 공식 문서는 MySQL 8.0 레퍼런스 매뉴얼

## 이 노트를 쓰게 된 상황

운영 중인 서비스의 테이블(설문 응답처럼 요청마다 행이 쌓이는 종류)에 날짜 컬럼 하나와 유니크 인덱스 하나를 추가하는 마이그레이션을 썼다. 처음엔 `ALTER TABLE ... ADD COLUMN ... AFTER ..., ADD UNIQUE KEY ...` 한 문장이었다. 코드 리뷰에서 세 가지 지적을 받았다 — `ALGORITHM=INSTANT`를 명시할 것, `AFTER` 절을 쓰지 말 것, 인덱스 추가를 별도 문장으로 분리할 것. 셋 다 "이대로 두면 배포 중에 테이블이 rebuild 된다"는 같은 이유였다.

이 문단에서 모르는 단어가 하나라도 있으면 이 문서가 도움이 된다. DDL이 무엇인지부터 시작한다.

## 질문 → 섹션

| 그때 떠오른 질문 | 답 |
|---|---|
| DDL이 뭔가, 왜 "온라인"이라는 말이 붙나 | [1장](#1-ddl과-온라인-ddl--용어부터) |
| "테이블 rebuild"가 물리적으로 무슨 일인가 | [3장](#3-rebuild란-무엇인가--새-파일을-만들어-전-행을-옮겨-적는-일) |
| `ALGORITHM=INSTANT`는 무슨 뜻인가 | [4장](#4-세-가지-알고리즘--copy-inplace-instant) |
| 행을 안 건드리면 그 컬럼 값은 어디서 오나 | [5장](#5-instant는-기존-행을-안-건드린다--그럼-읽을-때는-어떻게-되나) |
| 안 적으면 MySQL이 알아서 해 주지 않나 | [6장](#6-안-적으면-조용히-내려간다--그래서-굳이-적는다) |
| 인덱스 추가는 왜 문장을 따로 빼야 하나 | [7장](#7-인덱스는-왜-별도-문장으로-빼야-하나) |
| `AFTER`를 왜 안 쓰나 | [8장](#8-버전에-따라-달라지는-것--8012와-8029) |
| INSTANT를 무한정 써도 되나 | [9장](#9-instant에는-64번이라는-한도가-있다) |
| 그래서 마이그레이션을 쓸 때 뭘 확인하나 | [10장](#10-정리--마이그레이션-체크리스트) |

---

## 1. DDL과 온라인 DDL — 용어부터

**DDL(Data Definition Language — 데이터 정의어)** 은 데이터가 아니라 *구조*를 바꾸는 SQL이다. `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE`, `CREATE INDEX` 같은 것들이다. 반대로 `INSERT`/`UPDATE`/`DELETE`/`SELECT`는 구조가 아니라 데이터를 다루므로 **DML(Data Manipulation Language — 데이터 조작어)** 이라고 부른다.

옛날 MySQL에서 `ALTER TABLE`은 사실상 서비스 중단을 뜻했다. 구조를 바꾸는 동안 그 테이블에 쓰기를 할 수 없었기 때문이다. 테이블이 크면 수 분에서 수십 분씩 걸렸고, 그동안 애플리케이션은 그 테이블에 INSERT/UPDATE를 하지 못했다.

**온라인 DDL(online DDL)** 은 "구조를 바꾸는 동안에도 서비스가 그 테이블을 계속 읽고 쓸 수 있게 하는 방식"을 말한다. MySQL 5.6부터 점진적으로 도입됐고, 8.0에서 상당히 넓어졌다.

여기서 바로 주의할 점이 하나 있다. **온라인 DDL이라고 해서 "공짜"라는 뜻은 아니다.** 쓰기가 막히지 않을 뿐, 뒤에서는 여전히 테이블 전체를 복사하고 있을 수 있다. 그 차이를 가르는 게 다음 장부터 볼 "알고리즘"이다.

## 2. 테이블은 파일이고, 행은 그 안에 특정 모양으로 놓여 있다

왜 컬럼 하나 추가하는 게 비쌀 수 있는지 이해하려면 저장 방식을 알아야 한다.

MySQL의 기본 저장 엔진인 **InnoDB**는 테이블 하나를 보통 `.ibd`라는 파일 하나에 담는다(`innodb_file_per_table` 설정이 기본 ON). 그 파일 안은 **페이지(page, 기본 16KB)** 단위로 나뉘고, 각 페이지 안에 행들이 **정해진 물리적 레이아웃**으로 줄지어 들어 있다.

레이아웃이란 이런 것이다. "이 행의 첫 4바이트는 `id`, 다음 가변 길이 구간은 `pad`, 그다음은 ..." 하는 식으로, 어느 바이트가 어느 컬럼인지가 정해져 있다.

```
[페이지 #37]
  행1: | id=1 | pad="xxx..." | ← 여기까지가 한 행
  행2: | id=2 | pad="yyy..." |
  ...
```

여기에 컬럼을 하나 추가한다는 건 **행의 모양이 바뀐다**는 뜻이다. 순진하게 생각하면, 이미 저장된 모든 행에 새 컬럼 자리를 만들어 줘야 한다. 행이 100만 개면 100만 번이다.

바로 이 "순진한 방법"이 다음 장의 rebuild다.

## 3. rebuild란 무엇인가 — 새 파일을 만들어 전 행을 옮겨 적는 일

**테이블 rebuild(재구성)** 는 InnoDB가 다음 순서로 하는 작업이다.

1. 새 `.ibd` 파일을 하나 만든다(임시 이름으로).
2. 기존 테이블을 처음부터 끝까지 읽으면서, 각 행을 **새 레이아웃으로 바꿔서** 새 파일에 적는다.
3. 보조 인덱스(secondary index — 기본키가 아닌 인덱스)도 전부 새로 만든다.
4. 다 끝나면 파일을 바꿔치기하고 옛 파일을 지운다.

여기서 나오는 비용이 세 가지다.

**시간** — 데이터 전체를 한 번 읽고 한 번 쓴다. 행 수에 비례한다.
**디스크** — 작업 중에는 옛 파일과 새 파일이 동시에 존재한다. 순간적으로 그 테이블 크기만큼 여유 공간이 더 필요하다.
**I/O 부하** — 그동안 디스크가 바쁘다. 같은 서버의 다른 쿼리도 느려진다.

로컬 MySQL 8.0.46에서 직접 재 봤다. 컬럼 하나를 추가하는 똑같은 작업인데 방식만 다르게 했다.

| 테이블 행 수 | 메타데이터만 변경 | 테이블 rebuild |
|---|---|---|
| 30만 행 | 0.044초 | 0.906초 |
| 240만 행 | 0.047초 | 6.888초 |

왼쪽은 행이 8배가 되어도 시간이 그대로다(0.045초는 사실상 접속 오버헤드고, 실제 작업 시간은 그보다 훨씬 짧다). 오른쪽은 행 수에 비례해 늘어난다 — 행이 8배가 되니 시간도 약 7.6배가 됐다.

**이게 실전에서 왜 중요한가.** 요청마다 행이 쌓이는 테이블은 운영에서 수백만~수천만 행이 된다. 로컬에서 240만 행에 7초였다면 운영에서는 분 단위가 될 수 있고, 그동안 디스크 I/O가 튀어 무관한 API까지 느려진다. 마이그레이션은 보통 애플리케이션 기동 시점에 자동 실행되므로, "배포 버튼을 눌렀는데 서비스가 몇 분간 느려졌다"가 된다.

## 4. 세 가지 알고리즘 — COPY, INPLACE, INSTANT

MySQL은 `ALTER TABLE` 끝에 `ALGORITHM=` 으로 **어떤 방식으로 수행할지**를 지정할 수 있다. 세 가지가 있다.

| 알고리즘 | 하는 일 | 테이블 rebuild | 작업 중 동시 DML | 비용 |
|---|---|---|---|---|
| `COPY` | 새 테이블을 만들어 전 행을 복사 | 함 | **불가** (쓰기 막힘) | 행 수에 비례 |
| `INPLACE` | 원본 테이블 안에서 처리 | **연산에 따라 다름** | 대체로 가능 | 연산에 따라 다름 |
| `INSTANT` | 데이터 딕셔너리(카탈로그)만 수정 | 안 함 | 가능 | 테이블 크기와 무관, 일정 |

가장 헷갈리는 게 `INPLACE`다. 이름이 "제자리에서"라 공짜처럼 들리는데, **INPLACE도 rebuild를 할 수 있다.** "새 테이블을 만들어 복사하지 않는다"는 뜻일 뿐, 내부적으로 테이블을 재구성하는 연산이 여럿 있다.

실제로 측정해 보면 드러난다. 120만 행 테이블에 컬럼 하나를 추가할 때다.

| 방식 | 걸린 시간 |
|---|---|
| `ALGORITHM=INSTANT` | 0.043초 |
| `ALGORITHM=INPLACE` | 3.045초 |

같은 "컬럼 하나 추가"인데 INPLACE는 70배 넘게 걸렸다. INPLACE로 컬럼을 추가하면 rebuild가 일어나기 때문이다(MySQL 8.0.12에서 INSTANT가 도입되기 전에는 이게 최선이었다).

정리하면 이렇다. **"온라인 DDL이니까 안전하다"가 아니라, "어느 알고리즘으로 도느냐"가 안전을 결정한다.**

## 5. INSTANT는 기존 행을 안 건드린다 — 그럼 읽을 때는 어떻게 되나

여기가 개념의 핵심이다. INSTANT는 **이미 저장된 행을 단 한 줄도 수정하지 않는다.** 그런데 어떻게 새 컬럼이 생기나?

InnoDB는 **데이터 딕셔너리(data dictionary)** — 테이블의 구조를 적어 둔 내부 카탈로그 — 에 이렇게만 기록한다.

> 이 테이블에는 이제 `changed_date`라는 DATE 컬럼이 있다. 단, **이 시점 이전에 쓰인 행에는 그 값이 물리적으로 없으니**, 읽을 때 기본값(여기서는 NULL)으로 간주하라.

그래서 디스크의 기존 행은 예전 모습 그대로다. 나중에 그 행을 SELECT 하면 InnoDB가 "이 행은 컬럼 추가 이전 버전이구나" 하고 판단해서 기본값을 채워 돌려준다. 실제로 그 행에 값이 물리적으로 기록되는 건 **그 행이 다음에 UPDATE 될 때**다.

책에 비유하면 이렇다.

- **rebuild** — 모든 페이지에 빈칸을 만들려고 **책 전체를 새로 인쇄**한다.
- **INSTANT** — 앞표지에 "앞으로 이 항목이 추가된다. 기존 페이지에는 비어 있는 것으로 본다"고 **한 줄 적어 둔다**.

이래서 테이블이 아무리 커도 시간이 일정하다. 작업량이 행 수와 무관하기 때문이다.

그리고 이 구조 때문에 InnoDB는 **행 버전(row version)** 이라는 걸 관리해야 한다. "이 행은 몇 번째 구조 변경 이전에 쓰였는가"를 알아야 어느 컬럼을 기본값으로 채울지 판단할 수 있기 때문이다. 이 버전이 무한정 쌓일 수는 없는데, 그 한도가 9장에서 볼 64다.

## 6. 안 적으면 조용히 내려간다 — 그래서 굳이 적는다

`ALGORITHM=`을 아예 안 쓰면 어떻게 되나? MySQL이 알아서 가능한 방식 중 가장 가벼운 것을 고른다. 그러면 안 적어도 되는 것 아닌가?

문제는 **고를 수 없을 때 조용히 무거운 쪽으로 내려간다**는 점이다. 직접 확인해 봤다.

```sql
-- ① 알고리즘을 안 적은 경우
ALTER TABLE t ADD UNIQUE KEY uk_t (a, c1);
-- → 에러도 경고도 없이 성공한다. INSTANT 가 아니었다는 신호가 아무 데도 안 남는다.

-- ② 명시한 경우
ALTER TABLE t ADD UNIQUE KEY uk_t (a, c1), ALGORITHM=INSTANT;
-- → ERROR 1845 (0A000): ALGORITHM=INSTANT is not supported for this operation.
--    Try ALGORITHM=COPY/INPLACE.
```

①의 위험은 **개발자가 "즉시 끝날 것"이라고 믿은 채 배포한다**는 데 있다. 로컬에서는 테이블이 작아서 순식간에 끝나니 아무 문제가 없어 보인다. 운영에 올라가서야 rebuild가 돌고, 그때는 이미 늦었다.

②는 그 상황을 **실행 전에 실패**로 바꾼다. 마이그레이션이 깨지고 배포가 멈추지만, 테이블은 멀쩡하다.

그래서 `ALGORITHM=INSTANT`는 성능 지시가 아니라 **안전 단언(assertion)** 으로 읽는 게 맞다. 뜻을 풀면 이렇다 — *"이 변경은 반드시 즉시 끝나야 한다. 그럴 수 없는 상황이면 실행하지 말고 실패해라."*

같은 성격의 장치가 프로그래밍에도 많다. 테스트의 단언, 널 검사, 타입 체크가 다 "조용히 잘못된 상태로 가느니 여기서 멈춰라"다. DDL 버전이라고 보면 된다.

## 7. 인덱스는 왜 별도 문장으로 빼야 하나

여기서 반전이 하나 있다. **인덱스 추가는 그 자체로는 rebuild가 아니다.**

MySQL 8.0 공식 문서의 온라인 DDL 표를 보면, 보조 인덱스(유니크 포함) 추가는 `INPLACE`로 수행되고 "Rebuilds Table: **No**", "Permits Concurrent DML: **Yes**"다. 즉 테이블을 재구성하지 않고, 작업 중에도 읽기/쓰기가 가능하다.

실측도 같다. 120만 행 테이블에서:

| 작업 | 알고리즘 | 걸린 시간 |
|---|---|---|
| 컬럼 추가 | INSTANT | 0.043초 |
| 컬럼 추가 | INPLACE (rebuild) | 3.045초 |
| 유니크 인덱스 추가 | INPLACE (rebuild 아님) | 0.595초 |

인덱스 추가가 0.6초인 건 rebuild 때문이 아니라, 인덱스를 만들려면 데이터를 한 번 훑어서 정렬해야 하기 때문이다. rebuild(3초)보다 훨씬 싸다.

**그럼 왜 나누나?** 인덱스 추가가 문제여서가 아니라, **한 문장에 묶으면 그 문장 전체가 INSTANT로 실행될 수 없기 때문이다.**

```sql
-- 한 문장에 묶고 INSTANT 를 요구하면
ALTER TABLE big
  ADD COLUMN c_x DATE NULL,
  ADD INDEX ix_x (c_x),
  ALGORITHM=INSTANT;
-- → ERROR 1845: ALGORITHM=INSTANT is not supported for this operation.
```

INSTANT를 명시했다면 여기서 실패한다. 명시하지 않았다면 6장에서 본 대로 **조용히 INPLACE로 내려가고, 그러면 컬럼 추가가 rebuild가 된다.** 인덱스를 곁들이는 바람에 원래 0.04초면 끝났을 컬럼 추가가 3초짜리 테이블 재구성이 되는 것이다.

그래서 이렇게 나눈다.

```sql
-- 1) 컬럼 추가 — 즉시 끝나야 한다고 단언
ALTER TABLE events
    ADD COLUMN changed_date DATE NULL COMMENT '...',
    ALGORITHM=INSTANT;

-- 2) 인덱스 추가 — INPLACE, rebuild 없음, 동시 DML 가능
CREATE UNIQUE INDEX uk_events_owner_changed_date
    ON events (owner_id, changed_date);
```

### 7.1 나누면 생기는 대가 — 원자성

문장을 나누면 잃는 게 하나 있다. **MySQL의 DDL은 자동 커밋(autocommit)이다.** 문장 하나가 끝나면 바로 확정되고, 여러 DDL을 한 트랜잭션으로 묶어 롤백할 수 없다.

그래서 위 예시에서 2번이 실패하면 1번(컬럼 추가)은 이미 적용된 채로 남는다. Flyway 같은 마이그레이션 도구는 그 마이그레이션을 "실패"로 기록하고, 사람이 수동으로 정리(repair)해야 한다.

실무에서는 보통 감수한다. 새로 추가한 컬럼이 전 행 NULL이면 유니크 인덱스 생성이 실패할 이유가 거의 없기 때문이다(NULL은 유니크 제약에서 중복으로 치지 않는다). 다만 **"이 두 문장은 원자적이지 않다"는 사실 자체는 알고 있어야 한다.**

## 8. 버전에 따라 달라지는 것 — 8.0.12와 8.0.29

INSTANT는 처음부터 있던 기능이 아니고, 할 수 있는 범위가 버전마다 다르다. 공식 문서 기준으로 두 분기점이 중요하다.

| 버전 | 달라진 점 |
|---|---|
| **8.0.12** | `ADD COLUMN`에 `INSTANT` 도입. 이 시점부터 컬럼 추가의 기본 알고리즘이 INSTANT가 됐다(그 전에는 INPLACE). 단 **컬럼을 맨 뒤에만** 추가할 수 있었다. |
| **8.0.29** | 컬럼을 **임의 위치**에 즉시 추가할 수 있게 됐다. 즉 `AFTER some_column`으로 테이블 중간에 넣어도 INSTANT가 가능하다. |

`AFTER`가 여기 걸린다. 8.0.29 미만에서는 `AFTER`를 쓰는 순간 INSTANT가 불가능해져서, 명시했으면 실패하고 안 했으면 rebuild가 된다.

로컬 8.0.46에서는 `AFTER`를 써도 INSTANT가 잘 된다.

```sql
ALTER TABLE t ADD COLUMN c2 DATE NULL AFTER a, ALGORITHM=INSTANT;
-- → 성공 (8.0.29+)
```

그런데도 마이그레이션에서 `AFTER`를 빼는 게 좋은 이유는 **버전 의존성을 통째로 없애기 위해서**다. 로컬과 운영의 MySQL 마이너 버전이 같다는 보장이 없고, 특히 관리형 서비스(AWS Aurora MySQL 같은)는 엔진 버전 문자열이 기반 MySQL의 마이너 버전을 그대로 드러내지 않는 경우가 있다. `AFTER`를 안 쓰면 8.0.12 이상 어디서든 INSTANT가 된다.

대가는 **컬럼이 테이블 맨 뒤에 붙는 것**뿐이다. 컬럼 순서는 조회 결과에 영향을 주지 않는다(`SELECT *`의 출력 순서가 달라질 뿐인데, 애초에 `SELECT *`에 순서를 의존하는 코드가 잘못된 것이다).

## 9. INSTANT에는 64번이라는 한도가 있다

5장에서 본 "행 버전"이 무한정 쌓이지는 않는다. 공식 문서는 **한 테이블의 최대 행 버전을 64**로 정하고 있다. INSTANT로 컬럼을 추가하거나 삭제할 때마다 1씩 는다.

직접 세어 봤다. 빈 테이블에 INSTANT로 컬럼을 하나씩 추가하면서 카운터를 관찰했다.

```
추가 #1  성공 → TOTAL_ROW_VERSIONS=1
추가 #32 성공 → TOTAL_ROW_VERSIONS=32
추가 #63 성공 → TOTAL_ROW_VERSIONS=63
추가 #64 성공 → TOTAL_ROW_VERSIONS=64
추가 #65 → ERROR 4092 (HY000): Maximum row versions reached for table rv_lab/t.
           No more columns can be added or dropped instantly. Please use COPY/INPLACE.
```

현재 값은 이렇게 볼 수 있다.

```sql
SELECT NAME, TOTAL_ROW_VERSIONS
  FROM information_schema.INNODB_TABLES
 WHERE NAME = 'mydb/mytable';
```

한도에 걸려도 막다른 길은 아니다. **테이블을 한 번 rebuild 하면 카운터가 0으로 리셋된다.** rebuild 과정에서 모든 행이 최신 레이아웃으로 다시 쓰이므로 "예전 버전 행"이 사라지기 때문이다. 확인해 보면 이렇다.

```sql
OPTIMIZE TABLE t;   -- 내부적으로 테이블을 rebuild 한다
-- → TOTAL_ROW_VERSIONS = 0, 이후 INSTANT 추가가 다시 가능
```

**실전에서의 의미.** 한 테이블에 64번이나 컬럼을 즉시 추가·삭제하는 일은 흔치 않으므로 보통은 신경 쓸 필요가 없다. 다만 스키마를 자주 바꾸는 테이블이라면, 어느 날 갑자기 `ERROR 4092`로 마이그레이션이 실패할 수 있다는 걸 알아 두면 당황하지 않는다. 그때 필요한 건 계획된 rebuild(`OPTIMIZE TABLE` 또는 `ALGORITHM=INPLACE`로 강제) 한 번이고, 이건 테이블이 크면 오래 걸리므로 트래픽이 적은 시간대에 해야 한다.

## 10. 정리 — 마이그레이션 체크리스트

운영 중인 테이블에 `ALTER TABLE`을 쓸 때 확인할 것들이다.

**컬럼 추가라면** `ALGORITHM=INSTANT`를 명시한다. 실패하면 그 자리에서 알게 되는 편이 배포 후에 아는 것보다 낫다.

**`AFTER`를 쓰지 않는다.** 컬럼 순서를 포기하는 대신 버전 의존성이 사라진다.

**인덱스 추가는 별도 문장으로 뺀다.** 인덱스 자체가 비싼 게 아니라, 같은 문장에 있으면 컬럼 추가까지 rebuild로 끌려 내려가기 때문이다. 대신 두 문장이 원자적이지 않다는 걸 주석에 남긴다.

**컬럼 추가 외의 연산은 rebuild를 의심한다.** 타입 변경, 기본키 추가, `NULL` 허용 여부 변경 등은 대개 rebuild다. 공식 문서의 온라인 DDL 표에서 해당 연산의 "Rebuilds Table" 칸을 확인하는 게 가장 확실하다(부록 A.1).

**주석에 왜 그렇게 썼는지 남긴다.** 마이그레이션 파일은 한 번 쓰고 다시 안 보는 경우가 많아서, 다음 사람이 무심코 두 문장을 합치거나 `AFTER`를 되살리기 쉽다.

**테이블이 얼마나 큰지 먼저 본다.** 로컬에서 즉시 끝났다고 운영에서도 그런 게 아니다. rebuild 비용은 행 수에 비례한다.

---

## 부록

### A.1 연산별로 rebuild 하는지 확인하는 법

MySQL 공식 문서의 [InnoDB 온라인 DDL 연산 표](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html)가 기준이다. 연산마다 네 칸이 있다.

| 칸 | 뜻 |
|---|---|
| Instant | `ALGORITHM=INSTANT`가 가능한가 |
| In Place | `ALGORITHM=INPLACE`가 가능한가 |
| **Rebuilds Table** | 테이블을 재구성하는가 — **여기가 비용의 핵심** |
| Permits Concurrent DML | 작업 중에 다른 세션이 읽고 쓸 수 있는가 |

자주 쓰는 것들만 옮기면 이렇다.

| 연산 | INSTANT | Rebuilds Table | 동시 DML |
|---|---|---|---|
| 컬럼 추가 | 가능 (8.0.12+) | 아니오 | 가능* |
| 보조 인덱스 추가(유니크 포함) | 불가 | 아니오 | 가능 |
| 기본키 추가 | 불가 | **예** | 가능 |
| 컬럼 타입 변경 | 불가 | **예** | 대개 불가 |
| 컬럼 이름 변경 | 불가 | 아니오 | 가능 |

\* 단 **AUTO_INCREMENT 컬럼을 추가할 때는 동시 DML이 허용되지 않는다**(공식 문서 명시).

### A.2 INSTANT가 아예 불가능한 조건들

공식 문서가 열거하는 제약 중 실무에서 마주칠 만한 것들이다.

- INSTANT를 지원하지 않는 다른 `ALTER TABLE` 동작과 **한 문장에 섞으면** 안 된다 (7장의 상황).
- `ROW_FORMAT=COMPRESSED` 테이블.
- `FULLTEXT` 인덱스가 있는 테이블.
- 임시 테이블.
- 추가 후 행 크기가 최대 허용치를 넘는 경우.
- 행 버전이 64에 도달한 테이블 (9장).

### A.3 알고리즘과 잠금은 서로 다른 축이다

`ALTER TABLE`에는 `ALGORITHM=` 말고 `LOCK=` 절도 있다. `NONE` / `SHARED` / `EXCLUSIVE` / `DEFAULT` 중 하나를 지정해서 "작업 중 어느 수준까지 동시 접근을 허용할지"를 요구할 수 있고, 요구를 못 맞추면 역시 실패한다.

`ALGORITHM`이 "얼마나 무거운 작업인가"를 다룬다면 `LOCK`은 "그동안 남들이 접근할 수 있는가"를 다룬다. 둘은 연관되지만 같은 것은 아니다 — 예를 들어 보조 인덱스 추가는 rebuild를 안 하면서도(가벼움) 동시 DML을 허용한다.

이 노트에서는 `LOCK` 절은 실측하지 않았다. 필요해지면 A.1의 공식 표에서 "Permits Concurrent DML" 칸을 먼저 보고, 거기에 맞춰 `LOCK=NONE`을 명시적으로 요구하는 식으로 쓰면 된다.

### A.4 이 노트의 실험을 재현하는 스크립트

로컬 MySQL(도커 등)에 붙어서 그대로 실행하면 본문의 수치를 다시 만들 수 있다. 중립적인 실습용 DB를 쓴다.

```sql
-- 준비: 120만 행 테이블
DROP DATABASE IF EXISTS ddl_lab; CREATE DATABASE ddl_lab; USE ddl_lab;
CREATE TABLE big (id INT PRIMARY KEY AUTO_INCREMENT, pad VARCHAR(200));
INSERT INTO big (pad)
SELECT REPEAT('x',200) FROM information_schema.COLUMNS a, information_schema.COLUMNS b LIMIT 300000;
INSERT INTO big (pad) SELECT REPEAT('y',200) FROM big;
INSERT INTO big (pad) SELECT REPEAT('y',200) FROM big;
```

```bash
# ① 메타데이터만 — 테이블 크기와 무관하게 일정
time mysql ... ddl_lab -e "ALTER TABLE big ADD COLUMN c_i DATE NULL, ALGORITHM=INSTANT;"

# ② rebuild — 행 수에 비례
time mysql ... ddl_lab -e "ALTER TABLE big ADD COLUMN c_p DATE NULL, ALGORITHM=INPLACE;"

# ③ 인덱스 추가 — rebuild 는 아니지만 스캔+정렬 비용
time mysql ... ddl_lab -e "CREATE UNIQUE INDEX uk_big ON big (id, c_i);"

# ④ 한 문장에 섞으면 INSTANT 불가 (ERROR 1845)
mysql ... ddl_lab -e "ALTER TABLE big ADD COLUMN c_x DATE NULL, ADD INDEX ix_x (c_x), ALGORITHM=INSTANT;"
```

```bash
# ⑤ 행 버전 한도(64) 확인 — 빈 테이블에서
mysql ... -e "DROP DATABASE IF EXISTS rv_lab; CREATE DATABASE rv_lab;
              USE rv_lab; CREATE TABLE t (id INT PRIMARY KEY); INSERT INTO t VALUES (1);"
for i in $(seq 1 70); do
  out=$(mysql ... rv_lab -e "ALTER TABLE t ADD COLUMN c$i INT NULL, ALGORITHM=INSTANT;" 2>&1)
  [ -n "$out" ] && { echo "#$i 에서 중단: $out"; break; }
done
mysql ... -e "SELECT TOTAL_ROW_VERSIONS FROM information_schema.INNODB_TABLES WHERE NAME='rv_lab/t';"

# ⑥ rebuild 하면 카운터가 0으로 리셋된다
mysql ... rv_lab -e "OPTIMIZE TABLE t;"
mysql ... -e "SELECT TOTAL_ROW_VERSIONS FROM information_schema.INNODB_TABLES WHERE NAME='rv_lab/t';"
```
