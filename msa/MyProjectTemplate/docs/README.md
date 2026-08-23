# MyProjectTemplate 문서 허브

## 문서 위치와 책임

| 위치 | 역할 | 주소 |
|---|---|---|
| GitHub | 코드와 함께 버전이 고정되는 기술 정본 | <https://github.com/chanho4702/MyProjectTemplate> |
| Notion | 사람이 읽는 프로젝트 포털, 의사결정과 진행 현황 | <https://app.notion.com/p/MyProjectTemplate-3bda0d2516be8096a918cbbb430f95bf?source=copy_link> |
| Obsidian | 개인 지식 베이스용 Markdown 미러 | `msa/MyProjectTemplate` |

코드와 정확히 일치해야 하는 설정, 실행법, 아키텍처 계약은 Git의 `docs/`를 기준으로 한다. Notion은 문서 탐색과 협업의 입구로 사용하고, Obsidian은 같은 Git 커밋을 가리키는 읽기용 미러로 유지한다.

## 문서 지도

- [빠른 시작](quickstart.md)
- [권장 아키텍처와 용량 등급](architecture.md)
- [환경 전략](environments.md)
- [모듈 카탈로그](module-catalog.md)
- [처리량과 가용성 검증](capacity-testing.md)
- [컨테이너 빌드](container-build.md)
- [로드맵](roadmap.md)
- [검증 기록](verification.md)

## 변경 시 동기화 규칙

1. 코드와 같은 PR에서 Git 문서를 먼저 갱신한다.
2. PR 병합 또는 직접 푸시 후 Notion 프로젝트 허브를 같은 Git SHA 기준으로 갱신한다.
3. Obsidian `msa/MyProjectTemplate` 미러를 갱신하고 인덱스의 `source_commit`을 바꾼다.
4. 구현되지 않았거나 검증되지 않은 항목은 완료로 표시하지 않는다.
5. 특정 TPS와 가용성은 환경·SLO·부하 결과가 함께 있을 때만 기록한다.

Notion이나 Obsidian에서 기술 내용을 먼저 수정했다면, 다음 코드 변경 전에 Git 문서로 역반영해 세 위치가 갈라지지 않게 한다.
