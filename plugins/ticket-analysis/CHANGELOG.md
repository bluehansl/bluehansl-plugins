# Changelog — ticket-analysis

## claude-0.1.0 (2026-07-31)

- 최초 릴리스: 티켓 분석·운영 스킬
  - `ticket-workflow` — 티켓 선언·컨텍스트 로드(이슈 트래커 본문·댓글·연결 이슈)·도구 선택·단계별 승인 절차(7단계)·체크리스트 문법 (호스트 지침에서 이전)
  - `artifact-routing` — 산출물 경로 라우팅·브라우저 리뷰 문서/Export 규칙·제출 감시 (호스트 지침에서 이전)
  - `ticket-md-schema` — ticket.md 표준 스키마 (호스트 지침에서 이전)
  - `requirement-refinement` — 요건 정제 (superpowers brainstorming 이식)
  - `plan-writing` — 구현 계획 작성 (superpowers writing-plans 이식, TDD 실행 체인 제외)
  - `defect-analysis` — 버그 원인 진단 (superpowers systematic-debugging Phase 1~3 이식)
- 설계: 스킬이 티켓 프로세스의 단일 소스, 호스트 지침에는 트리거 문구만 잔존
- 검증: 가상 티켓 3종 with/without 벤치마크 100% vs 80%, 사용자 정성 리뷰 통과
