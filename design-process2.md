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

  # 북산 비서실 v1 — CP15 깃허브용 (2026-06-21)

> 아침 브리핑 파이프라인 6단계 전체 구현·실측·무인 자동화 — 전체 내용 + 결정 태그
> 직전 CP14(1순위 SOUL 배치 + 시간버그 해결 + Claude Code 설치) 이후 ~ 2순위 완성까지의 델타.
> 작업폴더 `~/buksan-briefing`. 진행방식 B(단계별 구현·실측). 봇 계정·호스트명·토큰·채널ID·IP는 마스킹.

---

## CP15. 아침 브리핑 파이프라인 — 6단계 전체 구현·실측·무인 자동화

### 설계 기조 [확정]

- **이한나 단일 창구 불변**: 이사장 ↔ 이한나만. 송태섭은 직보 안 함, 뒤에서 작성. 발송은 이한나 명의.
- **최대 토큰 효율**: 봇을 깨우지 않고 스크립트가 Claude API 직접 호출. 수집·발송은 스크립트(공짜), LLM은 작성·검증만 2콜.
- **호출 구조**: 스크립트가 송태섭→이한나를 API로 차례 호출(오케스트레이터=스크립트). CP4 "봇 위임"에서 변경 [승인].

### 1단계 — 시세 `market_fetch.py` [확정]

- Finnhub `/quote` 주력 → AV `GLOBAL_QUOTE` 폴백 → `unavailable`(값 안 지어냄).
- 8티커: 지수 SPY·QQQ·DIA + 워치 AMD·ROKT·GLW·DELL·ORCL. watchlist config 분리.
- KST 고정(timezone +9), `classify_trade_day()`로 휴장·주말 건너뛴 직전 거래일 판정. 장중 의심 시 `warning:intraday_possible` 별도 필드(status enum 안 깸).
- 키 `.env` 분리, 로그 키 비노출. 종료코드 키없음=3·전실패=4.
- **첫 실행 8/8 OK 전부 Finnhub(폴백 0), exit0.** ROKT 유효(close 117.58), Juneteenth(6/19)+주말 건너뛰고 trade_day=6/18.

### 2단계 — 뉴스 `news_fetch.py` [확정]

- AV News & Sentiment(시세 백업 AV키 재사용). topic 2개(economy_macro+financial_markets) 각 10건=20건, `sort=RELEVANCE`, 지난 24h.
- **relevance_rank 보존**(AV 관련도 topic별 1~10). 저장은 published_kst 최신순·rank 독립. note 메타로 "순서≠중요도, 선별은 내용 기준" 명시.
- 개별 종목 노이즈는 수집에서 안 거름 → 송태섭 몫(설계 원칙) [확정].
- **첫 실행 2/2 topic OK.** AV 무료 티어가 topic 필터 받아줌(관문 통과). 중복 제거 동작.

### 3·4·5단계 — `briefing_llm.py` (파이프라인 두뇌) [확정]

- 스크립트가 송태섭·이한나를 Claude API 차례 호출. 평시 LLM 2콜, 반려 시 최대 4콜.
- SOUL 실제 파일에서 읽음(`read_soul_core`): scout/main의 SOUL.md+IDENTITY.md+USER.md만(없으면 exit5 위조 방지). AGENTS/TOOLS/HEARTBEAT 제외.
- 모델 claude-sonnet-4-6, **thinking=disabled + effort=low**(adaptive 토큰 낭비 회피) [승인].
- **캐싱 끔** [승인]: 평시 cache_read=0이라 캐싱이 손해(+$0.004) → 기조 "최대 효율"에 맞춰 끔.
- 이한나 검증 = **structured output**(JSON schema passed/failed_criteria/reason)으로 게이트 견고화. 재작성 금지(토큰 절약, output 23토큰).
- **반려 = A안** [확정]: 1회 반려 후 재미달이면 "검증 미달 항목" 붙여 확정(무한 루프 금지).
- **거시 경계 = A** [승인]: 거시 부족 시 섹터 테마로 유연하게 채우되 `[섹터 신호]` 명시, 개별 종목은 섹터 근거로만(단독 시황 금지).
- **첫 실행**: 평시 2콜·이한나 1차 통과·exit0. 노이즈 17건 걸러냄(Dynamix·코카콜라·마카오 등 → AI 전환·반도체 랠리·AI 전력 인프라 3핵심). 수치 JSON 일치. ≈$0.076/회. 본문 우수(5칸·이사장 호칭·이모지 없음·한줄결론=상황 요약·투자 권유 없음·출처 표기).

### 6단계 — 발송 `send_briefing.py` [확정]

- 슬랙 `chat.postMessage` 직접. **이한나(Buksan Secretary) xoxb 토큰**(송태섭 아님). `auth.test`로 봇 신원 사전 확인.
- md→슬랙 mrkdwn 변환(헤더→`*굵게*`, `**`→`*`, 표→코드블록, `---`→빈 줄). **TABLE_STYLE 상수**로 codeblock↔lines 전환 가능(폰 깨짐 대비).
- 4000자 초과 시 스레드 분할. 실패 시 1회 재시도→실패 알림. `--notify-failure <단계>` 모드(마스터 호출용).
- **첫 발송 성공**: daily-briefing에 이한나 명의 5칸 브리핑 게시(PC·폰 OK).

### 7단계 — 마스터 + 크론 [확정]

- `run_briefing.sh`: 절대경로 `PROJECT=/Users/*****/buksan-briefing`·`PY=/usr/bin/python3`, KST 날짜, `run_step()`로 4단계 순차(market→news→briefing_llm "$DATE"→send_briefing "$DATE"). 앞 단계 실패 시 슬랙 알림+exit1 중단. 로그 output/logs. `set -u`. 손 테스트 ALL OK.
- **★크론 방식 = cron 확정(launchd 아님)** [확정 — 핵심 교훈]:
  - launchd `gui/501`은 **헤드리스 로그인 윈도우**(`/dev/console` 소유자 root)라 `StartCalendarInterval` 미발사. GUI 세션이 active해야 발사되는데 봇 맥은 헤드리스. gateway는 KeepAlive 상시 데몬이라 캘린더 발사 능력을 검증 못 해줬음(착각 유발).
  - 레거시 `load`·모던 `bootstrap` 둘 다 11:50·12:10 미발사 실측 확인 → **cron 전환**(시스템 cron 데몬, GUI 무관, sleep 0 확인됨).
  - cron 12:20 임시 테스트 → **실측 발사 확인** → `crontab "0 6 * * *"` + `SHELL=/bin/bash` + `PATH` 명시 프로덕션 확정.
  - **★교훈**: 헤드리스 봇 맥에선 GUI LaunchAgent 캘린더 안 됨 → cron 써야 함. (Claude Code MEMORY.md에 `macmini-headless-scheduling` 저장)
- plist `RunAtLoad=false`(등록 시 즉시 실행 방지), StandardOut/ErrorPath를 로그 파일로(봇 plist의 /dev/null과 달리 크론 실패 원인 보존).

### 키·설치 [확정]

- 사용자가 nano로 `.env` 직접 입력: `SLACK_BOT_TOKEN`(이한나)·`SLACK_CHANNEL_ID`·`ANTHROPIC_API_KEY`. `grep -c your_=0` 확인.
- **ANTHROPIC_API_KEY = 봇 키와 공유**($50 한도) [승인]: 봇·브리핑 둘 다 무인이라 같은 키. 코딩(Claude Code)만 구독으로 분리.
- `pip3 install anthropic`(user 설치).

### 비용 실측

- 1회 ≈ $0.074(input ~16.5K·output ~1.7K, 2콜). 월 1회/일 ≈ ~$2.2.
- ⚠️ 콘솔 Usage는 지연·집계 단위 때문에 1회를 정확히 못 떼어 보임(로그가 실측 진실). 봇 대화가 같은 키라 합산됨 → [TODO] 브리핑 전용 키 분리 검토 시 콘솔 추적 명확해짐.

### 완성 상태

**매일 06:00 KST 무인 자동: 시세 → 뉴스 → 송태섭 작성 → 이한나 검증 → 이한나 명의 슬랙 발송.**

### 남은 TODO

- output 누적 정리 정책(매일 JSON·로그 쌓임)
- 봇 키 $50 한도 재점검(브리핑 LLM도 같은 키, 월 ~$2) / 브리핑 전용 키 분리 검토
- 멘션 겹침 미세조정, 송태섭 응답 안정성 모니터
- 채소연 엔진, v2 6인 SOUL 파일화
- 보안(VNC 암호·Tailscale 2FA·토큰 SecretRefs), 슬랙 owner·allowlist 유저ID, 리포명

## CP16. 무인 자동화 실전 검증 + 다음 일감(CDA·권준호) 설계 + V2 방향

> 직전 CP15(아침 브리핑 6단계 구현) 이후 ~ 무인 검증·다음 일감·V2 방향까지의 델타.

### 1. ★무인 자동화 실전 검증 [완료]

- **2026-06-22T06:00:10+09:00 첫 자동 발사 확인** (슬랙 스크린샷 실측).
- 사람 개입 0. cron(`0 6 * * *`)이 헤드리스 맥미니에서 GUI 세션 없이 발사 → 4단계(시세→뉴스→작성→검증) 전부 통과 → 이한나(Buksan Secretary) 명의 발송.
- **★반려 루프 A안 실전 작동**: 송태섭 1차 작성 → 이한나 검증 반려 → 송태섭 재작성 → 통과. LLM 4콜(평시 2콜 + 반려 2콜). 무한 루프 안 빠짐(1회 반려 후 통과 = 설계대로).
- 본문 정직성 확인: 시세 6/18(trade_day) vs 뉴스 6/21(기사 기준) 기준일 상이를 송태섭이 명시 설명. 확인 불가 종목 0.
- **LOOP 판정**: "완료는 선언이 아니라 통과의 결과" — CP15는 선언, 이 자동 발사가 통과. 채점자 모드 최종 관문 통과. 2순위 아침 브리핑 **진짜 완료**.

### 2. 슬랙 봇은 시스템을 못 바꾼다 [실증 2회]

- **실증 A (일요일 주간요약)**: 이한나에게 "일요일 브리핑을 주간 요약으로" 지시 → (1)이한나→송태섭 직접 메시지가 권한 제한으로 막힘(OpenClaw 봇 간 통신 미설정 = CP4 토큰효율로 봇 위임 폐기) (2)슬랙 봇과 크론 스크립트(briefing_llm.py) 분리라 송태섭 "반영하겠다"가 빈 약속.
- **실증 B (7시 변경)**: 이한나에게 "내일 하루 7시로" 지시 → 이한나가 crontab 확인 시도했으나 "크론 잡 없다"고 **오보**(권한·컨텍스트 불일치로 빈 결과를 봄. 실제 cron은 존재). 송태섭에게 멘션 전달했으나 무의미.
- **★핵심 개념 확정**: 브리핑 쓴 송태섭(briefing_llm.py 안 API 호출, 새벽 크론)과 슬랙 송태섭(OpenClaw 상시 봇)은 **같은 SOUL 다른 실행** = 서로 모름·기억 공유 안 함. 아침 브리핑 슬랙 발송도 봇이 아니라 send_briefing.py가 이한나 토큰으로 직접 올림.
- **예측**: 내일(6/23) 브리핑은 cron `0 6` 그대로라 6시에 옴(봇 말과 무관). 실측 대기.

### 3. 다음 일감 확정 — CDA 상권분석 + 권준호 9인째 [별도 프로젝트로 분리]

- **일감**: WSL의 강력한 CDA(상권분석) 엔진을 ★jcw가 직접★ 맥미니로 마이그레이션(conda osx-arm64, 에이전트 불가=인프라 설치라 사람 일). localhost로 띄우면 권준호가 호출·해석. 대화형(온디맨드). 엔진=계산/데이터, 권준호=해석·번역가("강력한 엔진의 통역사").
- **권준호 페르소나** [확정]: 서랍의 Feynman 폐기 → 경제학자/현장형 상권 전문가로 교체. SOUL은 jcw가 작성(존재목적="권리금·보증금 날리지 않게 지키는 방패", 직설적, 금지어="대박날것같다/무난하다/잘될겁니다"). 도메인 지식=흐르는 입지 vs 머무는 입지·도깨비 상권 필터·3중 배후지(1차 500m 매출 60-70%·2차 1km·3차 광역)·손익분기 월세 공식(적정 임대료=목표 매출×10%, 12% 초과=노동 지옥). 출력=①한줄결론 ②입지 치명타 ③데이터 검증 ④액션 플랜.
- **빈틈 채운 결정**: (1)AGENTS=권준호 내부 3관점(현장 발품·데이터 분석·손익 계산), 별도 봇 아닌 단일 페르소나의 세 렌즈. (2)보고 라인=이사장 직접 대화(이한나 경유 안 함). 직보 예외가 안 감독·권준호 둘로 늘어남 → 추후 경계. (3)안전장치=A(직설 유지 + 면책 한 줄: "판단은 데이터·현장 기반 의견, 최종 계약 책임은 사장님").
- **채널 구조** [확정]: daily-briefing(아침 브리핑, 이한나·송태섭만) + 신규 showdown(끝장토론, 소문자 `showdown`, 이사장↔팀원 직접 대화·권준호 상권 상담). 채널=용도별, v2에서 도메인별 재분리(measure-first).
- **분리 조치**: 새 프로젝트 "CDA 마이그레이션"으로 분리. 시드 문서·권준호 SOUL 6파일 통합본 생성. 이 프로젝트(북산 비서실)는 **daily briefing 집중**으로 확정.
- 권준호 슬램덩크 인물 유지, 서태웅은 구현 라이벌(B안)로 그대로(직무 변경 안 함).

### 4. V2 목표 — 대화 기반 시스템 개선 ("분리하되 이어주기") [방향 확립]

- jcw 핵심 표현 **"서로 분리하지만 이어주기"**: 슬랙 대화 봇과 크론 스크립트는 분리(각자 효율) 유지하되, 슬랙 지시가 시스템에 반영되게 연결.
- 두 능력 필요: ▶능력1=봇 간 협업(이한나가 송태섭·강백호 등에 위임·수령, 멀티 에이전트). 단 토큰효율상 v2에서도 스크립트 오케스트레이션이 봇 간 직접 대화보다 효율적일 수 있음. ▶능력2(핵심)=봇이 시스템을 실제 변경.
- **구현 방향 비교**: 방식 A=슬랙 봇이 코드(briefing_llm.py) 직접 수정=위험(잘못 고치면 새벽 브리핑 깨짐). **방식 B(★권장)**=둘 사이에 "설정 파일 다리"(예 briefing_config.json) → 슬랙 봇은 코드 안 건드리고 설정(데이터)만 기록("일요일=주간요약"→config), 크론 스크립트는 돌 때마다 그 설정 읽어 실행. 분리 유지 + 이어주기 + 안전(봇이 로직 아닌 데이터만 바꿔 깨질 위험 적음).
- **★위험**: 봇이 자기 것 고치면 시스템 깨질 수 있음 → 안전장치 필수(검수·승인·롤백, 정대만 검수자 활용, 위험 결정 정지 원칙).
- **진행 계획**: V1(아침 브리핑) 안정화 후 바로 V2 착수.
- 즉시 TODO(V2 안 기다림): 일요일 주간 요약 변경 = 맥미니에서 briefing_llm.py 직접 수정.

### 5. 부가 결정

- **웹 검색 기능 후보**: 슬랙 대화 봇(이한나·송태섭)에 실시간 웹서칭 추가 검토. 방향 A=Claude API 내장 web_search(★권장, ANTHROPIC_API_KEY 그대로·키 추가 없음) / B=별도 검색 API(Perplexity·Brave) / C=Gemini(부적합, 구독≠API). OpenClaw가 내장 web_search 지원하는지 확인 후 A/B 확정.
- **슬랙 표시 이름 변경 시도 → 보류**: "Buksan Secretary"→한글 변경은 재설치 필요(토큰 리스크) 대비 이득 적음 → 이름 포기, 아이콘(이미지)으로 캐릭터 살리는 방향(재설치 불필요). 단 슬램덩크 원작 이미지는 깃허브·블로그 공개 시 저작권 이슈 → 오리지널 권장.
- **User OAuth Token(xoxp-) revoke 가능 확인**: 우리 시스템은 Bot Token(xoxb-)만 사용, xoxp-는 불필요 → 보안상 revoke 권장.

### 6. 현재 편성 (9인)

| # | 이름 | 엔진 | 역할 | 상태 |
|---|---|---|---|---|
| 1 | 이한나 | JARVIS | 비서실장·조율·검증·단일 창구 | v1 가동 |
| 2 | 송태섭 | The Wolf | 정보통·속공·브리핑 작성 | v1 가동 |
| 3 | 안 감독 | Carl Sagan | 리서치·직보 | v2 |
| 4 | 강백호 | Jack Sparrow | A안 구현 | v2 |
| 5 | 서태웅 | Spock | B안 라이벌 구현 | v2 |
| 6 | 정대만 | Bounty Hunter | 검수 | v2 |
| 7 | 채치수 | Master Chief | 보안 | v2 |
| 8 | 채소연 | 미정 | CSM | v3 휴면 |
| 9 | 권준호 | 경제학자/상권전문가 | CDA 상권분석 | 설계 완료(CDA 프로젝트) |

- 직보 예외: 안 감독(리서치) + 권준호(대화형 상담).

### 7. 남은 TODO

- 내일(6/23) 6시 브리핑 확인 — 봇 시스템 변경 불가 실증(7시 지시했으나 6시에 올 것).
- 일요일=주간 요약 변경(briefing_llm.py 직접 수정, 즉시 가능).
- 실패 알림(--notify-failure) 실제 작동 테스트 미실시.
- output 누적 정리 정책, 봇 키 $50 한도 재점검(브리핑 LLM도 같은 키·월 ~$2).
- 채소연 엔진, v2 6인 SOUL 파일화, 보안(VNC 암호·Tailscale 2FA·토큰 SecretRefs), 슬랙 owner·allowlist, 리포명.

