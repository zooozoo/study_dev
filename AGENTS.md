# study_dev — 개인 개발 학습 노트

이 저장소는 **개발 일반 지식**을 공부하고 정리하는 개인 공간이다. 특정 회사 프로젝트의 산출물이 아니다.

## 구조 — 카테고리 중심, 규모 따라 성장
- 탐색 단위는 파일이 아니라 **카테고리**다. `INDEX.md`가 유일한 진입점 — 카테고리 밑에 문서 섹션 링크를 단다.
- 규모 확장 규칙: 카테고리는 한 파일의 **섹션**으로 시작 → 쌓이면 **독립 파일** → 파일이 여럿이면 **폴더** 승격. 지금은 파일 단계다.
- 파일명은 주제 기준 `<주제-슬러그>.md` (티켓 키를 파일명에 쓰지 않는다 — 계기는 문서 머리에 한 줄로).
- **노트를 추가·개명·확장하면 반드시 INDEX.md의 해당 카테고리를 갱신한다.**

## 다른 컴퓨터에서 세팅
- 이 repo의 `.claude/skills/study-note/SKILL.md`가 **스킬 원본**이다. Claude Code는 repo 안에서 열면 자동 로드된다.
- repo 밖 어디서든 쓰려면(전역 설치) clone 후 아래를 실행:
  ```bash
  mkdir -p ~/.claude/skills/study-note ~/.codex/skills/study-note
  cp .claude/skills/study-note/SKILL.md ~/.claude/skills/study-note/
  cp .claude/skills/study-note/SKILL.md ~/.codex/skills/study-note/
  ```
- 스킬을 수정하면 repo 원본을 고치고 위 명령으로 재배포한다(전역 사본은 파생본).

## 노트 작성 기준
전역 `study-note` skill의 기준을 따른다(Claude Code·Codex 양쪽에 설치됨). 요약:
- 배경 지식 없이 읽을 수 있게 — 약어·전문용어는 첫 등장에서 바로 풀이, "당연히"로 건너뛰는 단계 금지.
- 도입에 "무엇을 왜 배우나" + (사건 기반이면) 한 문단 요약, 이어서 "질문 → 섹션" 표.
- 기초 → 심화 순서. 서술형 헤딩. 깊은 갈래는 부록으로.
- 표본: [realtime-web-networking-cors-infra.md](realtime-web-networking-cors-infra.md)

## ⚠️ 내용 경계 (개인 repo — 회사 자산 반입 금지)
- 회사 소스코드 원문 발췌, 내부 URL·호스트명, 시크릿·자격증명, 고객/환자 데이터, 미공개 제품 정보를 **넣지 않는다**.
- 회사 일에서 배운 개념은 **일반화해서** 쓴다 — 회사 코드 대신 최소 재현 예제, 실제 도메인 대신 중립 예시.
- 커밋·푸시는 사용자가 직접 한다(에이전트는 커밋 메시지만 제안).
