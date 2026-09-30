# 부산대 TECH WEEK 2026 문서

부산대 TECH WEEK 2026 개발·운영 문서를 찾는 첫 페이지입니다. 사람이 읽을 설명은 [human](human/README.md), AI에게 작업을 맡길 때 필요한 안내는 [ai](ai/README.md)에 있습니다.

| 필요한 문서 | 읽을 곳 |
|---|---|
| 개발·운영 중 찾아볼 내용 | [사람용 문서](human/README.md) |
| AI에게 줄 작업 규칙 | 해당 저장소의 `AGENTS.md`와 [AI용 문서](ai/README.md) |
| 새 문서를 쓸 때 | [문서 작성 안내](human/how-to/write-docs.md) |
| 같은 내용의 문서가 여러 개일 때 | [원본 문서 위치](human/reference/document-locations.md) |

## 폴더 구성

각 저장소는 별도의 Git 저장소입니다. 코드 저장소의 사람용 문서 원본은 해당 저장소의 문서 폴더에 있습니다.

```text
docs/
├── README.md          전체 문서 안내
├── human/             사람이 읽는 설명과 공통 규칙
├── ai/                AI 작업 안내와 참고할 파일 목록
└── docs.config.json   저장소·사이트 설정
PNU-TECHWEEK-260930/   AGENTS.md + docs/human + docs/ai
```

## 위키 사이트

사람용 문서는 [위키 사이트](https://tech-week-2026-kimepark.github.io/docs/)에 게시합니다. 게시 대상은 docs 저장소 `main`의 `human/`과 Intro 저장소 `kth`의 `docs/human/`입니다.

| 조건 | 반영 시점 |
|---|---|
| docs 저장소 `main` push | 즉시 빌드·배포 |
| Intro 저장소 `kth` 문서 변경 | 매시 17분 예약 빌드 |
| 수동 실행 | Actions의 Documentation website에서 Run workflow |

로컬에서 확인하려면 이 저장소 루트에서 다음 명령을 실행하십시오. Node.js 22 이상이 필요합니다.

```bash
npm install
npm run docs:build
npm run docs:preview
```
