# 원본 문서 위치

내용을 수정할 때 기준이 되는 원본 파일 목록입니다. 같은 설명이 여러 곳에 있으면 이 표의 원본을 수정하고 나머지는 원본으로 연결하십시오.

| 내용 | 원본 위치 | 담당 영역 |
|---|---|---|
| 문서 작성 기준 | `human/how-to/write-docs.md` | 전체 |
| 장애 조사 기록 | `human/explanation/troubleshooting/` | 조사한 담당자 |
| AI 작업 규칙 | 각 저장소의 `AGENTS.md` | 해당 저장소 |
| AI가 참고할 문서 목록 | `ai/sources.json` | 해당 작업 |
| 과거 보고서·초안 | `human/archive/` | 작성한 담당자 |
| 팀 역할·Git·협업 규칙 | `human/reference/team/` | A |
| intro 문서(강의 자료) | `PNU-TECHWEEK-260930/docs/human/`, `kth` 브랜치 | intro |
| sar-robot 개발 문서 | `sar-robot/docs/human/` | 변경한 모듈 담당자 |
| 코드 작성 규칙, 모듈 인터페이스 | `sar-robot/docs/human/reference/` | A, 인터페이스 제공 모듈 담당자 |

프로젝트에 맞게 행을 추가하십시오. 구체적인 담당자는 팀에서 배정합니다.

## 저장소별 문서

- [intro 문서](https://github.com/Tech-Week-2026-KimEPark/Intro/blob/kth/docs/human/README.md)
- [sar-robot 문서](https://github.com/Tech-Week-2026-KimEPark/sar-robot/blob/main/docs/human/README.md)

## 위치 고정 파일

`README.md`, `AGENTS.md`, `CLAUDE.md`는 사람과 도구가 가장 먼저 찾는 파일이므로 저장소 루트에 있어야 합니다. 라이선스와 에셋 출처는 관련 파일과 같은 위치에 있어야 합니다.

## 문서와 코드 불일치 시 기준

현재 동작의 근거는 해당 버전의 코드·설정·테스트입니다. 팀이 의도한 규칙은 작업 지침과 정책 문서에서 확인하십시오. 두 내용이 다르면 그 차이를 수정하거나 논의하십시오. 확인하지 않은 운영 상태는 미확인 정보로 취급하십시오.
