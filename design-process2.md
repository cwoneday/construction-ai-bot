# CP14 — 1순위 SOUL 배치 + 멀티봇 + 속도/안정성/시간 결정
**날짜:** 2026-06-21
**선행:** CP13(VNC+Tailscale 외부접속)
**상태:** v1 가동 2인(이한나·송태섭) 슬랙에서 실전 동작. 브리핑 자동화(2순위) 직전.

---

## A. 1순위 — 이한나 SOUL 배치 [확정]

### 발견: workspace 기본 파일 충돌
봇 생성 시 OpenClaw가 기본 파일을 깔아둠. 우리 6파일 패키지와 이름이 겹침.
| 파일 | 처리 | 근거 |
|---|---|---|
| SOUL.md (1806B 기본인격) | **교체** | 순수 인격, OpenClaw 동작 무관 |
| IDENTITY.md | **교체** | 동일 |
| USER.md | **교체** | 동일 |
| AGENTS.md (8109B) | **`>>` 이어붙이기** | OpenClaw 운영매뉴얼(세션·메모리·도구 규칙). 통째 교체 시 봇 동작 깨짐 |
| soul.json | **신규 생성** | OpenClaw 기본엔 없음 |
| TOOLS.md / HEARTBEAT.md / MEMORY.md | **안 건드림** | OpenClaw 소유 |

### 방법 [확정]
- `unset zle_bracketed_paste` (붙여넣기 `^[[200~` 깨짐 방지) 선행 — 필수.
- `cat > 파일 << 'CLAWEOF'` heredoc. 끝 표지를 `CLAWEOF`로(내용 중 EOF 충돌 방지).
- AGENTS는 `cat >> AGENTS.md`(이어붙이기). 검증=`head`(원본 살았나)+`tail`(우리 섹션 붙었나).
- soul.json은 `python3 -m json.tool`로 문법 검증(한글 `\u…` 이스케이프는 정상).
- 백업 먼저(`cp 파일 파일.bak`), 한 파일씩 `cat`으로 검증하며 진행.

### 검증 [확정]
슬랙 자기소개 → 이사장 호칭·비가역3종·리뷰게이트·슈퍼바이저강약(부서대상한정)·이모지없음·단일창구·리마인드 전항목 통과.

---

## B. 자잘한 TODO [확정]
- `rm ~/.openclaw/workspace/*.bak`
- 게이트웨이 토큰 rotate: `openclaw doctor --generate-gateway-token` (127.0.0.1 전용이라 노출 위험 낮음, config get 시 `__REDACTED__`로 가려짐).
- **한국 위기번호** [확정, 웹검색]: 자살예방 상담전화 **109**(24시간, 2024년 1393 등 통합·1393 폐지), 정신건강 상담전화 **1577-0199**, 긴급 **112/119**. `sed -i ''`로 SOUL.md 위기자원 줄 교체.

---

## C. 송태섭(Buksan Scout) 멀티봇 [확정]

### 명명 규칙 [확정]
"Buksan" 접두어 + 역할: Buksan Secretary(이한나), Buksan Scout(송태섭). v2도 동일.

### 슬랙 2번째 앱 (5함정 선제 회피)
Socket Mode→xapp / Bot Token Scopes 6개(chat:write·channels:read·channels:history·groups:read·groups:history·app_mentions:read) / Event Subscriptions 4개(app_mention·message.groups·message.channels·message.im)+Save / Install→xoxb / 채널에 "Add apps to this channel"로 초대(/invite는 봇 안 뜸).

### OpenClaw 멀티 에이전트 등록 [확정]
문서(docs.openclaw.ai/concepts/multi-agent): accountId(슬랙앱)+agentId(인격, 별도 workspace+agentDir+세션)+binding(라우팅). **agentDir 절대 공유 금지**(auth/세션 충돌). auth는 에이전트별(`~/.openclaw/agents/<id>/agent/auth-profiles.json`).

`openclaw agents add scout` 위저드 흐름:
- workspace=`~/.openclaw/workspace/scout`(main과 분리, 기본값 OK)
- "Copy portable auth profiles from main?" = **Yes**(키 복사 — agentDir 공유 아님, 안전)
- "Configure model/auth now?" = No
- 채널 설정 → Slack → "Install Slack plugin?"에서 **Skip하면 계정등록 경로가 막혀 도로 나옴** → Download from npm 또는 "Modify settings"로 진행
- Slack account → **"Add a new account"**(default=이한나 건드리지 않기)
- account id = `scout`
- bot token "Keep it?" 나오면 **No** → Buksan Scout xoxb 직접 입력 → xapp 입력
- channels allowlist = `C0BBUT8ERFX`(ID, 이름 빼기)
- DM 정책 = No / 바인딩 = Yes
→ `Agent "scout" ready`. (백업 `openclaw.json.bak` 자동 생성)

### SOUL 6파일 배치 [확정]
이한나와 동일 방식 + **scout엔 BOOTSTRAP.md 존재 → `rm BOOTSTRAP.md`**("나 누구지" 모드 원인). 검증: 송태섭 자기소개에서 이름·정보통·이사장·확인불가·리뷰게이트·이모지없음 통과.

---

## D. 멀티봇 라우팅 디버깅 [확정] — 멀티 에이전트 핵심 난관

문서가 "중복응답으로 고생, 리셋해야 풀림" 경고한 영역. 순차 해결:

1. **겹침**(scout 부르면 이한나도 답함): scout만 명시 바인딩, 이한나는 default fallback이라 모든 메시지 수신.
   → `openclaw agents bind --agent main --bind slack:default` (둘 다 명시).
   → bindings 2줄: `scout←slack:scout`, `main←slack:default`. 겹침 해소.

2. **이한나 무응답**: 단일→멀티 전환 시 위저드가 `accounts.default`에 채널 설정을 안 채움(scout엔 있는데 default엔 없는 비대칭).
   → `openclaw config set channels.slack.accounts.default.channels.C0BBUT8ERFX.enabled true`.

3. **여전히 멘션 무응답**: 로그 `[slack-auto-reply] {reason:"no-mention"} skipping channel message`. 멀티 계정에서 이한나 봇이 자기 멘션을 "no-mention"으로 오판정.
   - app_mention 이벤트는 있음(가설 기각). tool-policy(coding 프로파일이 message 도구 제거)도 scout는 같은데 답하므로 단독 원인 아님.
   → `requireMention false`로 복귀(`config set channels.slack.accounts.default.channels.C0BBUT8ERFX.requireMention false`). 이한나 응답 복구.

4. **송태섭 일시 무응답**(다음날): 죽은 게 아니라 게이트웨이 일시 다운이었고 **KeepAlive가 새벽 자동 복구**. 송태섭 자가진단으로 확인(추정 명시). 구조 정상.

**[검증] 둘 다 실전 동작:** 실제 경기 스코어를 BBC 출처와 함께, "추정"은 추정으로 명시해 응답. SOUL 절대규칙("출처 없는 숫자 금지·확인불가는 확인불가로") 실전 작동 확인. ⚠️ 데이터 정확성 자체는 별도 검증 필요(출처 표기 ≠ 정확성 보장).

---

## E. 속도·안정성·시간 — 브리핑 자동화 토대

### 안정성 [확정]
`~/Library/LaunchAgents/ai.openclaw.gateway.plist`: `KeepAlive=true`(죽으면 자동 부활), `RunAtLoad=true`(부팅 시작), `ThrottleInterval=10`(crash loop 방어). 어제 무응답=일시 다운, 자동 복구가 보장됨.
- [관찰] `StandardErrorPath=/dev/null` — 크래시 원인이 안 남음. 무응답 반복 시 로그 파일로 변경 가능(현재 미적용).

### 속도 [확정]
- 원인: thinking 미설정 → Claude 4.6 기본값 `adaptive`가 단순 질문에도 사고 → 지연.
- `openclaw config set agents.defaults.thinkingDefault low` → 재시작 → 빨라지고 품질 유지.
- ⚠️ per-agent thinkingDefault는 config 스키마에서 거부될 수 있음(이슈) → **전역(agents.defaults)으로** 설정.
- fast모드(`agents.defaults.models["anthropic/claude-sonnet-4-6"].params.fastMode`): Anthropic service_tier auto에 매핑되나 **Priority Tier 계정 아니면 standard로 떨어져 무효** → 보류. (모델 키의 `/` 때문에 config set 꼬일 위험도 있음)
- 모델: sonnet 유지(opus는 alias만 등록, 깊은 분석 담당 생기면 그때). 필요시 메시지에 `/think high`로 일시 상향.

### 시간 [확정]
브리핑 **06:00 KST 고정**. 장 마감: 서머타임 05:00 KST·표준시 06:00 KST. 기상 시간 우선(데이트레이딩 아님). 겨울엔 마감 직후라 일부 "확인불가" 가능하나 SOUL이 빈칸 안 채움. ⚠️ 맥미니 시간대가 PDT(-07:00)일 수 있음 → 크론 작성 시 KST 변환 확인 필수.

---

## 미해결 [TODO]
- **★2순위 브리핑 파이프라인**: 크론 06:00 KST → 시세수집(Finnhub→AV·ETF프록시) → 뉴스수집 → 송태섭 선별·해설·§11 5칸 → 이한나 검증(리뷰게이트) → 스크립트 발송. LLM은 선별·해설·검증만, 수집·발송은 스크립트(비용절감). **시작 전 시세 API키(Finnhub·AV) 발급 확인 필요.** Claude Code 작업.
- 멘션 겹침 미세조정(이한나 requireMention false라 scout 부를 때 낄 수 있음 — 봇 user ID 매칭 정밀화 필요)
- 송태섭 응답 안정성(일시 무응답 재발 모니터)
- 봇 Anthropic 지출한도 재점검(현재 $50, 무인 데몬)
- 채소연 엔진 확정, v2 6인 SOUL 파일화
- 보안: VNC 강한암호·Tailscale 2FA·토큰 SecretRefs 이전
- 슬랙 owner 등록·allowlist에 유저ID(doctor 권고)
- GitHub 리포명 확정
