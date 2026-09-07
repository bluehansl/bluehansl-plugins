# multi-check 장애 기록 — 리뷰어 전원 timeout으로 결과 0 (2026-09-07)

> 발생 세션: FASSTO EYE 에이전트 자동 업데이트 구현 플랜 검토 (백그라운드 job 세션)
> deft 버전: `claude-2.50.0` · 환경: **orca**
> 결과: 스킬의 정규 경로(Agent 리뷰어)로는 **양쪽 엔진 모두 결과를 얻지 못함**. Lead가 CLI를 직접 실행해 우회.
>
> **📌 처리 상태 (2026-09-07 갱신)**: **P0 3건 + P1 2건 반영 완료** → `claude-2.51.0` / `codex-1.26.0`.
> 설계 근거는 `RATIONALE.md` **R-18** 로 이관됐다(이 문서는 원본 실측 기록으로 보존).
> P2(detach)는 미착수 — `PENDING.md` 참조. §6 우회 절차는 이제 불필요하나 참고용으로 남긴다.

---

## 1. 한 줄 요약

리뷰어 페르소나의 **Bash timeout 120초**가 실제 CLI 소요시간(3~10분)보다 짧아 의미 있는 검토는 **구조적으로 항상 timeout**되고, 그 뒤 Lead의 per-report `shutdown_request`가 백그라운드로 넘어간 CLI까지 거둬 **거의 완성된 분석이 폐기**된다.

## 2. 타임라인 (실측)

| 단계 | 결과 |
|---|---|
| Phase 2 preflight | `CODEX_OK` / `CLAUDE_OK` / `DEFT_ENV=orca` / PERSONA_DIR = marketplace 경로 — **정상** |
| Agent spawn ×2 (claudex-reviewer, claude-reviewer) | 정상. orca 가드대로 cmux/tmux 캡처·`cmux-rebalance-watch` **skip** — 정상 |
| claudex-reviewer가 `deft-review codex` 실행 | **120초 timeout** → Bash가 백그라운드로 자동 이동 → 페르소나 규율대로 "timeout 사실" 보고 |
| Lead가 per-report `shutdown_request` 발송 | 리뷰어 graceful 종료. **동시에 백그라운드에서 돌던 codex 프로세스도 종료** — 출력 파일 말미가 `[killed]`, 분석 본문 0 |
| claude-reviewer | 동일하게 120초 timeout, 부분 출력조차 없음 |
| Lead 우회 1차 (`cd ~`에서 직접 실행) | **exit 1**. 출력 파일에는 `[exited with code 1]`만 남아 원인 불명 |
| 원인 재현 | `Not inside a trusted directory and --skip-git-repo-check was not specified` |
| Lead 우회 2차 (git 리포에서 실행) | **성공** — 두 엔진 모두 완전한 검토 결과 획득 |

폐기된 claudex 출력 8,416바이트의 내용은 **프롬프트 에코 + web search 로그 11줄**이 대부분이었고, 분석 본문은 시작 전이었다.

## 3. 근본 원인

### R1. 페르소나 timeout(120초)과 실제 소요시간의 구조적 불일치
- 페르소나 명세: `deft-review ... (Bash, timeout 120000)`
- 실제: `gpt-5.5` + `xhigh` reasoning으로 6KB 프롬프트를 검토하면 **3~6분**, web search가 붙으면 **10분 이상**
- 결과: **짧은 질문에서만 동작하고, 스킬의 본래 용도(설계·코드 교차검증)에서는 항상 실패**한다
- 페르소나가 `run_in_background` 금지를 명시(노이즈 방지)하고 있어 리뷰어에게 우회 수단이 없다

### R2. timeout 보고를 "결과 보고"로 취급해 shutdown이 작업을 죽인다
- 스킬 Phase 4는 "report 1건 수신 → 즉시 그 리뷰어에게 `shutdown_request`"를 지시한다
- 그러나 timeout 보고는 **결과가 아니라 실패 보고**인데 Lead에게 이를 구분할 분기가 없다
- shutdown이 리뷰어 프로세스를 내리면서 그 자식(백그라운드 CLI)까지 함께 종료된 것으로 보인다 — 발송 직후 출력이 `[killed]`로 끝남(정황 근거)
- 리뷰어가 결과 파일 경로를 함께 보고했음에도, 그 파일이 완성되기 전에 프로세스가 사라졌다

### R3. `deft-review`가 git 리포 밖에서 동작하지 않음
- claudex의 trusted directory 검사에 걸려 `exit 1`
- **스킬·페르소나 어디에도 이 전제가 없다.** Lead가 우회 실행할 때 cwd가 리포 밖이면 즉시 실패
- 더 나쁜 점: 에러 메시지가 출력 파일에 남지 않고 `[exited with code 1]`만 기록돼 원인 파악에 추가 시도가 필요했다

### R4. time-box 지침이 claudex에 효과 없음
- 스킬 Phase 1이 요구하는 문구("빠른 교차검증 — 신속히 진행하고… 과도한 다중 web search는 하지 말 것")를 프롬프트에 포함했음에도 **web search 11회** 수행
- 프롬프트를 **"웹 검색 절대 금지"** 로 바꾸자 즉시 준수했고, 2차 검토는 수 분 내 완료
- 완곡한 권고형 문구는 이 엔진에 통하지 않는다

## 4. 정상 동작한 부분 (오해 방지)

**orca pane 조작은 실패 원인이 아니다.** 이번 사건에서 orca 관련 동작은 전부 설계대로였다:
- `ORCA_*` 환경변수로 `DEFT_ENV=orca` 정확히 판정
- orca 가드대로 `cmux identify`/`tmux list-panes` 캡처와 `cmux-rebalance-watch` 발사를 **전부 skip**
- 페르소나 인라인(marketplace 경로 우선) 정상
- 두 리뷰어 모두 `shutdown_response`로 graceful 종료, **orphan pane 없음**
- `SendMessage` 보고 규약(`summary` 포함) 정상

실패는 **CLI 실행 계층(timeout·trusted directory·검색 폭주)** 에서 났다.

## 5. 재현 조건

- deft `claude-2.50.0`, orca(또는 cmux) 환경
- 검토 프롬프트가 수 KB 이상이고, claudex가 web search를 시작할 만한 주제(인프라·보안·플랫폼 사실 확인이 필요한 설계 검토)
- → 리뷰어 2개 모두 120초에 걸려 결과 0

## 6. 이번에 사용한 우회 절차

정규 경로가 막혔을 때 Lead가 직접 수행한 순서 — 재발 시 그대로 쓸 수 있다:

1. Agent 리뷰어를 쓰지 않고 **Lead가 CLI를 직접 백그라운드 실행**
   ```bash
   cd ~/git/<아무 git 리포>            # trusted directory 필수 (R3)
   deft-review codex  < prompt.md      # run_in_background: true
   deft-review claude < prompt.md      # run_in_background: true
   ```
2. 프롬프트 말미에 **강한 금지형 제약**을 넣는다 (R4)
   ```
   웹 검색 절대 금지 — 아는 지식만으로 즉시 결론. 5분 내 완성. 서론 재서술 금지.
   ```
3. 완료 알림을 받아 결과 파일을 Read

이 방식은 timeout 제한이 없고(백그라운드), 두 엔진 모두 완전한 결과를 냈다.

## 7. 개선 제안 (우선순위 순)

| # | 대상 | 제안 | 반영 |
|---|---|---|---|
| P0 | 페르소나 `agents/*.md` | `deft-review` 실행의 Bash timeout을 **600000(10분)** 으로 상향. 현 120초는 스킬의 주 용도에서 항상 실패한다 | ✅ 2.51.0 |
| P0 | SKILL.md Phase 4 | **timeout 보고와 결과 보고를 구분**한다. timeout 보고를 받으면 shutdown을 보류하고, 리뷰어가 함께 보고한 출력 파일이 완성될 때까지 Lead가 대기(백그라운드 감시)하도록 분기 추가 | ✅ 2.51.0 — 첫 줄 센티널(`TIMEOUT_PARTIAL`/`RESULT`/`FAILED`) 분기 + Error Handling 표 동반 수정 |
| P1 | `deft-review` 헬퍼 | cwd가 git 리포 밖이면 **명확한 메시지로 안내**하거나 `--skip-git-repo-check`를 자동 부착. 최소한 stderr를 출력 파일에 남겨 `[exited with code 1]`만 남는 상황을 없앤다 | ✅ 2.51.0 — `--skip-git-repo-check` 자동 부착(리포 밖 실행 실측 확인). ※ stderr 지목은 정정 — 헬퍼의 codex/claude 경로는 stderr 를 버리지 않았고, 실제 결함은 **gemini 경로의 `2>/dev/null`**(제거) 과 우회 절차의 `2>&1` 누락이었다 |
| P1 | SKILL.md Phase 1 | time-box 지침을 **금지형**으로 교체 — "과도한 web search는 하지 말 것"(무력) → "웹 검색 금지(필요 시 1~2회 상한)" | ✅ 2.51.0 |
| P2 | 페르소나 | 리뷰어가 CLI를 부모와 분리해(detach) 띄워, 리뷰어 종료 후에도 결과 파일이 완성되게 하는 방안 검토 (R2의 대안 해법) | ⏸ 미착수 — `PENDING.md` 등재 |

## 8. 영향

- 이번 세션은 우회로 목적을 달성했다(1차·2차 검토 모두 완전한 결과 확보)
- 다만 **정규 경로만 따르면 결과가 0**이므로, 스킬을 처음 쓰는 세션은 "리뷰어가 전부 timeout" 상태에서 멈출 가능성이 높다
- ~~P0 2건을 반영하기 전까지는 §6 우회 절차를 사용하는 편이 안전하다~~ → **P0·P1 반영 완료(2.51.0)로 정규 경로 복구.** §6 은 참고용으로만 남긴다
- **보고서가 놓쳤던 것 (수정 시 발견)**: ① Error Handling 표의 `Timeout … | Synthesize with available results` 행이 폐기를 **명시적으로 지시**하고 있었다 — Phase 4 만 고치면 이 행이 수정을 무효화한다 ② Codex 포팅본도 폴링 예산이 동일하게 120초(`seq 1 60`×`sleep 2`)라 같은 이유로 항상 미완료였다 — 단 포트는 pane 독립 프로세스 + `tee` 라 조기 종료 사고는 나지 않는다
