<!-- ticket.md 표준 스키마의 단일 소스 문서. 호스트 지침에서 스킬로 이전(2026-07-31, verbatim). -->

# `ticket.md` 표준 구조

`~/.ai/tickets/<TICKET_ID>/ticket.md` 의 표준 스키마. 단일 Claude·팀 작업 공통.

```markdown
# {TICKET_ID} - {제목}

## META
- 티켓: {TICKET_ID}
- 상태: {Draft | In Progress | Blocked | 검증중 | 완료}
- 작성일 / 최종 수정일: {YYYY-MM-DD}
- 브랜치: {feature/TICKET_ID}
- 대상 메뉴/모듈: {메뉴명 (모듈코드)}
- 핵심 파일: `{파일경로}` ({메서드/쿼리id}, 라인 {N~M})
- 연결 이슈: (relates to 가 있는 경우)

## 요건 분석
### 작업 목표 / 작업 범위(포함·제외) / 요구사항 상세 / 원인 분석(버그·튜닝)

## 영향도 확인
### Controller / Service / Mapper · SQL / Entity · 기타

## 설계 결정
### 채택한 방식 / 미채택 대안 및 사유
### 미해결 결정사항
- [ ] {Q1} → 해결 시 [해결] {결론, YYYY-MM-DD}

## 작업 계획
<!-- 체크리스트가 유일한 진행 상태 소스. 의존: ← (N번과 병렬) / ← (N번 후속) -->
### Phase 1 DB/Mapper · Phase 2 Backend · Phase 3 Frontend · Phase 4 검증

## 협의사항
- 역할 간 인터페이스 협의·기획 변경 이력

## FRONTEND / BACKEND / QA / REVIEW
- 각 역할 작업노트 취합본 + 코드 리뷰·테스트 결과

## 검증 이력
### {YYYY-MM-DD} {제목} — 목적 / 방식 / 결과 / 잔여물

## 롤백/복구
- DB 롤백: {SQL 또는 절차} / 코드 롤백: {브랜치·커밋 해시}
```

- 체크리스트 플래그·상위 상태 종합·넘버링·의존 표기 문법은 `ticket-workflow.md` §5를 따른다.
