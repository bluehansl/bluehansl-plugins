# deft — 진행 중·보류 항목 (PENDING)

> 진행 중이거나 미뤄둔 작업·이슈를 추적한다. 완료되면 CHANGELOG 로 옮기고 여기서 제거.
> 설계 근거는 `RATIONALE.md`, 버전 이력은 `CHANGELOG.md`.

상태 플래그: `[ ]` 미착수 · `[>]` 진행중 · `[!]` 보류(사유) · `[x]` 완료(→ CHANGELOG 이관)

---

## 진행 중

- [>] **🔴 multi-check L4 페르소나 준수 검증 (claude-2.51.0 잔여)** — 2026-09-07 착수 · **새 세션에서 재개**
  - **왜 남았나**: 2.51.0 의 메커니즘(timeout→background→완주)은 실측 PASS 했으나, **리뷰어 Agent 가 새 센티널 절차를 실제로 이행하는지**는 확인 못 했다. 원인은 코드가 아니라 환경 — 장수 orca 세션(49분+)에서 `Failed to create teammate pane: Timed out waiting for the Orca runtime to respond` 로 **Agent spawn 전면 차단**(deft-test 함정 #23 계열).
  - **⚠️ 전제**: **새 세션 초반에 즉시 실행**할 것(함정 #23 예방 — 세션이 길어지면 또 막힌다).
  - **재현 절차**:
    ```bash
    WT=~/orca/workspaces/bluehansl-plugins/deft-fix        # 또는 현 워크트리
    # (1) 신버전 헬퍼 스테이징 (캐시는 아직 2.50.0 — 배포 전이라 수동)
    cp ~/.local/bin/deft-review /tmp/deft-review.BACKUP
    cp $WT/plugins/deft/bin/deft-review ~/.local/bin/ && chmod +x ~/.local/bin/deft-review
    # (2) timeout 을 60초로 낮춘 테스트 페르소나 (경로를 결정적으로 강제 — deft-test 함정 #27)
    sed 's/timeout 600000/timeout 60000/; s/10분 timeout 을/60초 timeout 을/' \
      $WT/plugins/deft/skills/multi-check/agents/codex-reviewer.md > /tmp/codex-reviewer-TEST60.md
    ```
    (3) 위 페르소나를 **Agent prompt 에 인라인**해 `codex-reviewer`(model haiku) spawn. 검토 과제는 3분 이상 걸릴 만한 설계 비교 질문 + "웹 검색 절대 금지".
    (4) 검증 후 원복: `cp /tmp/deft-review.BACKUP ~/.local/bin/deft-review`
  - **PASS 기준** (하나라도 어긋나면 페르소나 문구 수정):
    - [ ] 60초 timeout 후 리뷰어가 **첫 줄 `TIMEOUT_PARTIAL`** 로 중간 보고 (평문 서술 아님)
    - [ ] 보고에 **출력 파일 경로** 포함 (`BashOutput` 을 호출하지 않을 것 — deprecated, 함정 #25)
    - [ ] Lead 가 `shutdown_request` 를 **보내지 않는** 동안 리뷰어가 출력 파일 `Read` 로 완료를 이어받음
    - [ ] 완료 후 **첫 줄 `RESULT`** 로 재보고 → 그때 Lead 가 shutdown → `shutdown_response` 로 graceful 종료
    - [ ] 중간 경과를 반복 보고하지 않음(노이즈 규율 준수)
  - **참고**: 실행 실패 경로(`FAILED` 센티널)는 gemini 리뷰어로 확인 가능 — 현 환경의 gemini 는 `IneligibleTierError`(계정 티어)라 **항상 실패**하므로 FAILED 보고 여부를 덤으로 볼 수 있다.


- [>] **SKILL.md 정리 — 테스트성·로그성·일지형 주석 걷어내기** (2026-06-25 착수)
  - 목표: 산재한 deft-log 호출 간소화(출력 레지스터 정책은 유지), 일지형 실측 주석을 RATIONALE.md 로 이관 후 SKILL.md 는 간결한 지침+`(근거: R-N)` 참조만, deft-test 연계·임시 잔재 제거.
  - **multi-round 1차 완료**: `RATIONALE.md` 신설(R-1~R-14) + 핵심 일지형 6건 RATIONALE 참조 축약 + deft-test L4 연계 제거. 실측 언급 39→32.
  - **남은 작업**:
    - [ ] multi-round 산재 deft-log 호출 17개 → 핵심 마일스톤만 간소화(§출력 레지스터 정책은 유지)
    - [ ] multi-round 짧은 인라인 실측 언급(32건) 중 불필요한 것 추가 정리
    - [ ] agent-teams SKILL.md 정리(deft-log 10·실측주석 13) — RATIONALE 에 항목 추가하며
    - [ ] multi-check SKILL.md 정리(실측주석 13)

## 보류 / 대기

- [ ] **multi-check 리뷰어 CLI detach — 리뷰어 종료와 CLI 수명 분리** (2026-09-07 등재 · P2)
  - **배경**: 2026-09-07 사고(RATIONALE R-18)에서 리뷰어 Agent 종료가 background 로 밀린 CLI 까지 함께 죽였다. `claude-2.51.0` 은 **센티널 분기로 "죽이지 않게"** 해결했지만(Lead 가 shutdown 을 보류), CLI 수명이 여전히 리뷰어 프로세스에 묶여 있다는 구조는 그대로다.
  - **제안**: `deft-review` 가 CLI 를 `setsid` 등으로 부모와 분리해 띄우고 출력을 파일로 tee → 리뷰어가 언제 종료돼도 결과 파일이 완성된다. Codex 포트가 이미 이 성질을 갖고 있다(pane 독립 프로세스 + tee) — 사고가 Claude 측에서만 난 이유.
  - **미착수 사유**: ① 2.51.0 의 센티널 분기로 실사용 재발은 막혔다(우선순위 하락) ② detach 는 페르소나에 raw bash 를 노출하거나 `deft-review` 에 출력파일 규약을 새로 만들어야 해 **표면 변경 범위가 P0 대비 크다** ③ 고아 프로세스 정리 책임이 새로 생긴다(현재는 리뷰어와 함께 정리됨).
  - **착수 판단 기준**: 20분 상한(리뷰어 이어받기)을 넘기는 검토가 실제로 반복되면 착수.


- [x] **🔴 Phase 3-A 재작성 — 회의 워커 헬퍼 기반 2채널 공존 복원** (2026-06-25 해결 → claude-2.43.0) — 2.40.0 회귀(빈 pane + CLI 직접부팅으로 `--claude-team-agent` binding 누락 → 이름표 사라짐 + cmuxKnock 폴백)를 R-16 절차(첫 워커 Agent tool + 나머지 헬퍼 `DEFT_BUS_DIR` 주입)로 교체. NTP binding(이름표·ntpPush) + 버스 board 2채널 공존 복원. `--inbox` register 가 ntpPush 스위치. 용례 1 단일 소스 통합·dangling 참조 정정 동반. 근거 RATIONALE R-16. **검증 대기**: 사용자가 다른 워크스페이스 날것 실행으로 확인(bash 직접 실측 금지 — R-8 맹점).

- [x] **테스트 세션 워커 부팅 실패** (2026-06-25 해결 → claude-2.42.0) — zsh 환경 cmux send 3대 함정(입력창 잔여·긴 명령 유실·colon 파싱). 사용자 세션 자체 보정 패턴을 일반화(C-u 클리어·source 파일화·명시적 나열). 근거 RATIONALE R-15.

- [!] **harness 방법론 부분 차용(C) — deft 문서/페르소나 흡수** (2026-06-30 회의 합의 · 구현 보류 — 사용자 승인 대기)
  - **근거**: `HANDOFF-harness-analysis.md` 의뢰 → multi-round 회의(dialogue, 3관점 mix: 하네스 아키텍트·deft 메인테이너·실무 사용자 대리) **3인 CONSENSUS**. 회의록 `~/.claude/plugin-data/deft/multi-round/sessions/standalone/20260630-171108-harness-deft/`(summary.md·transcript.md·board.jsonl).
  - **결론**: harness를 새 스킬 신설(A 내재화)·상시 병용(B) 하지 않고, **알짜만 기존 문서/페르소나에 명문화 흡수**. 코드 0줄·새 스킬 0개·PATCH급.
  - **흡수 작업 (승인 후 착수)**:
    - [ ] `skills/multi-round/SKILL.md` — 회의 모드 참가자 구성(Phase 1-4 페르소나 결정 절차)에 harness **B 에이전트 분리 4축**(전문성/병렬성/컨텍스트/재사용성 — 2축↑ 강하면 분리) 짧은 체크리스트/reference 로 흡수.
    - [ ] `skills/multi-round/SKILL.md` — **작업 모드(Phase 4-T)**에 harness **A 6대 토폴로지 미니 매트릭스**(pipeline/fan-out/expert-pool/producer-reviewer/supervisor/hierarchical) 각주. **회의 모드엔 미적용**(회의=팬인 수렴 단일 패턴이라 6대 토폴로지는 잡음).
    - [ ] `skills/agent-teams/SKILL.md` — 고정 8역할 **유지**, 토폴로지는 참고 각주만.
    - [ ] 릴리즈 체크리스트(또는 RATIONALE/GUIDE) — skill description 변경 시 harness **C 트리거 검증**(should-trigger 8~10 + near-miss 8~10) 편입.
  - **하지 않는 것(기각)**: (A) 페르소나 생성 보조 스킬 신설 — 단일 플러그인 유지비 영구반복 + no-overlink 위반 + "언제 뭘 쓸지" 4번째 결정축 혼란. (B) harness 상시 병용 — 빌드타임 메타툴 vs 런타임 일회성 시점 부정합 → 예외/고급 흐름 문서화만.
  - **불변식**: deft 정체성 3축(이종 AI 연계·work-id 영속·본업 컨벤션 강제) **무손상** — 흡수는 "기존 Lead 판단을 어휘로 보강"까지만. 균형추 memory `feedback_no_overlink_deft`.
  - **1차 분석 수정 반영**: A(6대 패턴) [상]→**[중]**(설계 렌즈로만, 런타임 엔진 아님). 실효 우선순위 **B ≳ C > A**.

- [ ] **모델 지정 전면 변수화 — `deft-model` SSOT 단일화 (멀티체크 haiku 제외)** (2026-07-02 가능여부 판단 완료 · 구현 대기)
  - **판단**: 가능. 모델 지정이 2종 — ① **CLI 실행**(`claude --model`)은 이미 `deft-model` 참조(일부 리터럴만 잔존) → `$(deft-model claude)` 로 완전 자동화 가능 / ② **Agent tool `model:`**(팀원·첫워커, alias `fable` ~10곳)은 도구 호출이라 `$()` 런타임 치환 불가 → "SSOT 참조" 관례로만 단일화.
  - **제약**: Agent enum 은 alias(`fable`)만, CLI 는 풀 ID(`claude-fable-5`)도 수용 — 두 표기 공존이라 deft-model 이 둘 다 내보내야 함.
  - **설계 (승인 시)**:
    - [ ] `deft-model` 에 alias 차원 추가: `deft-model claude`→`claude-fable-5`(CLI), `deft-model claude alias`→`fable`(Agent enum). 양 포트. (모델 교체 시 여기 2값만)
    - [ ] CLI 리터럴 → `$(deft-model claude)`: `bin/deft-claude-native-spawn` 기본(`${4:-fable}`), multi-round 헬퍼 호출 인자. 완전 자동 전파.
    - [ ] Agent tool 문서 ~10곳 축약: 단일 선언 1곳 + 나머지(agent-teams SKILL/GUIDE·`agents/*.md` 8종·multi-round 첫워커)는 그 선언 참조. SKILL 이 "Lead 는 spawn 시 `deft-model claude alias` 로 모델 결정" 명시.
  - **범위 제외**: multi-check `model:"haiku"`(의도된 경량 래퍼) — 변수화 대상 아님.
  - **트레이드오프**: 장점 = 모델 교체가 진짜 1곳(deft-model). 단점 = Agent tool 문서 자기완결성↓(reader 가 deft-model 조회 필요) + Lead 가 spawn 때 SSOT 조회하도록 SKILL 강제 필요. 작업량 ≈ claude-2.45.0 Fable 변경과 유사(양 포트).
  - **동기**: claude-2.45.0 Fable 복원 때 opus→fable 를 양 포트 ~20곳 sed 로 교체했음 — 이 변수화로 차기 모델 교체는 deft-model 1곳으로 축소.
