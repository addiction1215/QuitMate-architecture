# QuitMate Architecture

[인터랙티브 아키텍처 다이어그램 열기](https://addiction1215.github.io/QuitMate-architecture/)

QuitMate 서비스의 시스템 구성과 흡연 기록 데이터 흐름을 비개발자도 탐색할 수 있게 정리한 정적 아키텍처 문서입니다.

## 문서

- `site/index.html`: 배포되는 문서의 시작 페이지
- `site/diagrams/`: 브라우저에서 바로 열 수 있는 독립 실행형 다이어그램
- `sources/`: 다이어그램 재생성·수정용 Archify JSON 원본
- `docs/`: 구성 요소와 판단 근거를 설명하는 설계 문서
- `evidence/`: 생성 당시의 시각 검증 결과

## 배포

`master` 브랜치에 변경 사항이 push되면 GitHub Actions가 `site/` 디렉터리를 GitHub Pages에 배포합니다.

GitHub Pages의 **Build and deployment → Source**는 **GitHub Actions**로 설정합니다.

## 업데이트 절차

1. 백엔드/프론트 구조 변경을 반영해 Archify 다이어그램 원본과 HTML을 갱신합니다.
2. `sources/`, `site/diagrams/`, 필요 시 `docs/`와 `evidence/`를 함께 갱신합니다.
3. Pull Request 검토 후 `master`에 병합합니다.
