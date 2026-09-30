# docs 작업 규칙

<!-- devdog-docs:begin docs-layout -->
## 문서 위치

| 내용 | 위치 |
|---|---|
| 사람용 설명 | `human/` |
| AI 작업 안내 | `ai/` |
| 문서 작성 기준 | [문서 작성 안내](human/how-to/write-docs.md) |

사람용 설명과 AI 안내에 같은 제품 규칙을 복사하지 마십시오. AI 안내는 사람용 원본 문서로 연결합니다.
<!-- devdog-docs:end docs-layout -->

<!-- devdog-docs:begin docs-update -->
## 기능 변경 시 문서 갱신

기능 추가·수정 작업의 완료 조건에는 관련 문서 갱신이 포함입니다. 별도 요청을 기다리지 말고 같은 작업에서 수행하십시오.

1. 작업 전에 [문서 작성 안내](human/how-to/write-docs.md)의 작성 기준과 원본 위치를 확인하십시오.
2. 변경한 API·화면·도메인·설정·운영 절차를 관련 사람용 원본 문서에 반영하십시오. 같은 설명을 AI 문서에 복사하지 마십시오.
3. 경로·명령·작업 규칙이 바뀌면 `AGENTS.md`와 관련 AI 안내·참고 파일 목록도 고치십시오. `CLAUDE.md`는 `AGENTS.md`를 참조하는 파일입니다.
4. `python3 scripts/check_docs.py --write-catalog`와 `python3 scripts/check_docs.py`를 실행하십시오. 형제 저장소가 있으면 `python3 scripts/check_docs.py --workspace`도 실행하십시오.
5. 완료 보고에 갱신한 문서와 검사 결과를 포함하십시오. 문서 변경이 불필요한 경우에는 이유를 적으십시오. 접근 제한으로 갱신하지 못했다면 미완료 항목으로 보고하십시오.
<!-- devdog-docs:end docs-update -->

<!-- devdog-docs:begin incident -->
## 장애 기록

배포된 서버·앱에서 확인된 오류를 수정하는 작업의 완료 조건에는 장애 기록이 포함입니다. 원인이 자명한 오타·문구 수정은 제외입니다. 수정과 같은 작업에서 수행하십시오.

1. `human/explanation/troubleshooting/`에 `YYYY-MM-DD(제목).md` 파일을 추가하십시오. 같은 폴더 `README.md`의 목록에도 한 줄을 추가하십시오.
2. 같은 원인이 다시 발생하면 새 파일 대신 기존 기록에 재발 날짜를 추가하십시오.
3. 구성은 해당 폴더 `README.md`의 "작성 방법"을 따르십시오.
<!-- devdog-docs:end incident -->
