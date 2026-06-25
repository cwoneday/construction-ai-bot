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

# CP17 — 아침 브리핑 주간 모드(일요일/월요일) 추가

**날짜:** 2026-06-22
**선행:** CP15(아침 브리핑 6단계 무인 자동화 완성), CP16(무인 검증 + CDA/권준호 설계)
**상태:** 3파일 코드 수정 완료 + 정적 검증 + weekday 통합 실행 검증 통과. sunday/monday end-to-end LLM 실행은 미수행(다음 세션).
**작업 폴더:** `~/buksan-briefing` (`/Users/*****/buksan-briefing`)

---

## A. 목표와 설계 결정

### A-1. 무엇을 추가했나
평일(화~토) 아침 브리핑은 현행 일별 등락 유지. 한국시간 일요일·월요일 새벽에 "주간 모드"를 추가.

- `[확정]` 일요일 = 이번 주 회고, 월요일 = 다음 주 전망
- `[확정]` 시세 = 주간 등락(금→금). 정확히는 "각 주 마지막 거래일 종가" 기준
- `[확정]` 뉴스 = 3모드 공통 지난 24h (현행 유지)
- `[확정]` 모드 3개: `weekday`(화~토) / `sunday` / `monday`

### A-2. 과거 종가 API 선택 — AV TIME_SERIES_DAILY 단독
현행 시세 수집(Finnhub `/quote`, AV `GLOBAL_QUOTE`)은 "최신 스냅샷 + 전일"만 제공. 특정 과거 날짜 종가를 못 줌.

- `[승인]` 과거 종가는 AV `TIME_SERIES_DAILY`(outputsize=compact) 단독 사용. Finnhub `/stock/candle`은 안 씀(무료 플랜 US 주식 candle 403 가능성, 키 등급 확인 회피)
- `[추론]` AV 무료 한도 = 일 25콜 + 분당 5콜. 주간 모드 날 = 뉴스 2콜 + 과거종가 8콜 = 10콜 < 25 (여유). 평일은 weekly 조회 0콜 (가드 밖)

### A-3. "지난주 금요일" 정의 — 주 마지막 거래일, 휴장 자동 흡수
- `[승인]` 주간 등락률 = (이번주 마지막거래일 close − 지난주 마지막거래일 close) / 지난주 close × 100
- `[승인]` "마지막 거래일" = 일별 시계열을 ISO 주차로 그룹핑 → 각 주 max(날짜). 금요일 휴장 시 목요일이 자동 선택됨(값 위조 없음, `classify_trade_day` 정신)

### A-4. §칸 구성 (평일 5칸 골격 유지, 포커스만 변경)
- `[확정]` 일요일(회고): 주간 시장요약 / 이번주 핵심흐름 3 / 주간 주목종목 / 잔존 리스크 / 한줄 회고
- `[확정]` 월요일(전망): 지난주 마감요약 / 다음주 관전포인트 3 / 주시종목 / 예상 리스크 / 한줄 전망
- `[확정]` 월요일만 "★전망 가드" 섹션 추가: "확인된 일정·이미 발표된 실적/이벤트 일정만 근거, 추측·예언 금지, 모르면 '확인된 일정 없음'으로". 일요일(회고)은 과거라 추측 위험 덜해 절대규칙에만 녹임
- `[확정]` 주시종목 = "워치리스트 중"(현행 5종목 기준). 섹터 ETF(SMH/XLF 등) 추가는 watchlist 재설계라 별도 작업으로 보류 (길1=쉽게 가기)

### A-5. 회귀 안전 원칙 (최우선)
- `[확정]` 무인 시스템이므로 평일 경로 1바이트도 안 바뀜이 1순위
- `[확정]` 모든 신규 로직은 `if weekly_mode` / `if mode in ("sunday","monday")` 가드 안에만. weekday는 가드를 안 탐
- `[확정]` 인자 없거나 weekday면 폴백 → 기존 동작

---

## B. market_fetch.py 수정

### B-1. 상수 추가
```python
AV_DAILY_URL = "https://www.alphavantage.co/query"  # TIME_SERIES_DAILY
WEEKLY_MODES = ("sunday", "monday")
AV_DAILY_SLEEP_SEC = 15  # 분당 5콜 제한 회피
```

### B-2. 신설 함수
- `_last_trading_day_per_week(series)` — 일별 시계열을 ISO 주차(`isocalendar()[:2]`)로 그룹핑, 각 주 max(날짜) = 마지막 거래일. 최신 주 먼저 반환. 휴장 자동 흡수
- `fetch_weekly_closes(ticker, av_key)` — AV `TIME_SERIES_DAILY` 단독. `this_week`/`last_week`(date·close)·`change`·`change_pct`·`source`. 실패/2주 미만/0 나누기 → `None`(위조 안 함). `Note`/`Information`(레이트리밋) → `RuntimeError`

### B-3. main() 배선
- `mode = sys.argv[1] if len(sys.argv) > 1 else "weekday"`, `weekly_mode = mode in WEEKLY_MODES`
- `if weekly_mode:` 안에서만 8종목(indices+watchlist) weekly 조회. 콜 간 `time.sleep(15)`(`i > 0` 가드 = 첫 콜 대기 없음, 8콜 ≈ 105초, 무인이라 무방)
- `output["mode"]` 항상 추가, `output["weekly"]`는 주간 모드만. 기존 `indices`/`watchlist` 블록 불변

### B-4. 검증
- `[확정]` ast 통과
- `[확정]` ★SPY 손 검산: last 2026-06-12(741.75) → this 2026-06-18(746.74), change_pct = (746.74−741.75)/741.75×100 = 0.6727 코드값과 일치
- `[확정]` ★Juneteenth(6/19 금) 휴장 → 목요일(6/18) 자동 흡수 실데이터 입증
- `[확정]` weekday 회귀: 구조 차이 0, 소요 5.5초(sleep 안 탐 = 평일 영향 0)
- `[추론]` sunday 첫 테스트 시 8종목 중 4종목만 성공(QQQ/AMD/GLW/ORCL는 분당 5콜 레이트리밋 unavailable). 버그 아님(설계대로 값 위조 안 하고 unavailable) → sleep으로 해결

---

## C. briefing_llm.py 수정

### C-1. SCOUT_TASK → SCOUT_TASKS dict (편집 1)
- `SCOUT_TASK`(단일 문자열) → `SCOUT_TASK_WEEKDAY`(변수명만 교체, 본문 불변) + `SCOUT_TASK_SUNDAY` + `SCOUT_TASK_MONDAY` 신설
- `SCOUT_TASKS = {"weekday": ..., "sunday": ..., "monday": ...}`
- `[확정]` weekday 본문 byte 동일 검증: 백업 `SCOUT_TASK` 1144자 == 현재 `SCOUT_TASK_WEEKDAY` 1144자, diff 0줄

### C-2. call_scout (편집 2)
- 시그니처에 `mode="weekday"` 추가(`reject_reason` 앞, 기존 호출이 키워드/미전달이라 순서 안 깨짐)
- `task = SCOUT_TASKS.get(mode, SCOUT_TASK_WEEKDAY)` — 오타/미지정 폴백
- 주간 모드면 user에 "weekly 필드(금→금 주간 등락) 기준" 한 줄 추가
- `system=_system_blocks(soul_core, task)` — 기존 `SCOUT_TASK` 참조를 `task`로 교체 → NameError 해소(단독 `SCOUT_TASK` 출현 0)

### C-3. call_hanna 당일자 완화 (편집 3)
- 시그니처에 `mode="weekday"` 맨 끝 추가
- `task = HANNA_TASK; if mode in ("sunday","monday"): task += 보정문`
- 보정문: "'당일자' 기준은 '주간 데이터 허용'으로 해석. weekly 수치 기반이면 당일자 미달로 반려 말 것. 수치출처·무모순은 엄격히"
- `[확정]` ★`CRITERIA` 리스트·`VERDICT_SCHEMA` enum의 "당일자"는 불변(byte 동일). 보정은 프롬프트 텍스트(`task`)로만 → 구조화 출력 스키마 무손상. VERDICT_SCHEMA는 CRITERIA 변수 참조(리터럴 아님)

### C-4. main() 배선 (편집 4)
- `mode = sys.argv[2] if len(sys.argv) > 2 else "weekday"` (argv[1]=date, argv[2]=mode)
- 호출부 4곳에 `mode=mode` 키워드 전달: 콜1 작성(L397) · 콜2 검증(L402) · 콜3 재작성(L412) · 콜4 재검증(L417)

### C-5. 검증
- `[확정]` 매 편집 ast 통과, import OK
- `[확정]` weekday 회귀: mode 폴백 → 4개 호출부 weekday → SCOUT_TASK_WEEKDAY + HANNA_TASK 원본, weekly 안내/보정문 안 붙음 = 평일 동작 불변

---

## D. run_briefing.sh 수정

### D-1. 요일 판정 → MODE (편집 3-a)
```bash
DOW="$(TZ=Asia/Seoul date +%u)"   # 1=월 … 7=일
case "$DOW" in
  7) MODE="sunday" ;;
  1) MODE="monday" ;;
  *) MODE="weekday" ;;
esac
MODE="${MODE_OVERRIDE:-$MODE}"     # 테스트 훅
```
- `[추론]` 토요일(date +%u=6)은 `*` → weekday (한국 토요일 새벽 = 미국 금요일 마감, 평일과 동일)

### D-2. 호출부 인자 전달 (편집 3-b)
- `run_step market_fetch.py "$MODE"`, `run_step briefing_llm.py "$DATE" "$MODE"`. news_fetch는 인자 없이 그대로
- `run_step` 정의(`$1`=스크립트, shift 후 `"$@"`)는 수정 불필요

### D-3. SKIP_SEND 훅 (편집 3-c)
```bash
if [ "${SKIP_SEND:-0}" = "1" ]; then
  log "SKIP_SEND=1 → 발송 단계 건너뜀 (테스트 전용)"
else
  run_step send_briefing.py "$DATE"   # 6단계: 슬랙 발송
fi
```
- `[확정]` ★프로덕션 안전: SKIP_SEND 미설정 → 기본값 0 → `"0"="1"` 거짓 → else → 정상 발송. launchd 실전엔 영향 0. 테스트만 `SKIP_SEND=1`로 차단. 주석 방식(되돌리기 까먹으면 실전 발송 안 됨)보다 안전
- `[확정]` `send_briefing.py` 자체는 안 건드림. run_briefing.sh에서 호출만 조건부

### D-4. 검증
- `[확정]` bash -n 통과, `run_step send_briefing.py` 1개(else 안, 중복 없음 — grep -cn 1)
- `[확정]` 발송 분기 시뮬레이션: 미설정→발송 / =0→발송 / =1→건너뜀

---

## E. weekday 통합 실행 검증 (실제 LLM 호출)

`SKIP_SEND=1 MODE_OVERRIDE=weekday ./run_briefing.sh`

- `[확정]` 끝까지 에러 없이 완주 (market_fetch → news_fetch → briefing_llm → 발송 건너뜀 → ALL OK)
- `[확정]` mode=weekday 로그(market_fetch·briefing_llm 둘 다), weekly 키 없음(has_weekly: False)
- `[확정]` SKIP_SEND=1 발송 차단(슬랙 안 건드림)
- `[확정]` 평일 5칸 정상("[섹터 신호]" 표기 포함), 이한나 1차 통과(2콜 종료, 반려 루프 안 탐)
- `[확정]` LLM 2콜(input 16256, output 1394) ≈ $0.07 (평소 비용 부합)
- **결론:** 정적 검증(ast/grep/bash -n) + 실행 검증 둘 다로 평일 경로 무손상 실증

> 참고: 같은 날 06:00 launchd 프로덕션 실행 로그도 별도로 보임(4콜, 당일자 반려 후 통과) — 테스트(23:48)와 무관한 실전 무인 실행. 평일 시스템 정상 가동 중.

---

## F. 작업 패턴·교훈

- `[확정]` measure-first: market_fetch가 과거 종가 낼 수 있는지 먼저 확인 후 설계 진입
- `[확정]` 단계별 절차: 백업(0단계) → 한 편집씩 적용 전 텍스트 미리보기 → 적용 → ast/bash -n → 회귀 확인 → 다음
- `[확정]` ★Claude Code str_replace가 old_string을 좁게 잡아 "기존 줄 안 지우고 추가" 중복 위험 반복(logger·sleep 루프·send 줄). diff 뷰어 디스플레이 잘림/중복 표시 빈번 → 매 편집 적용 후 `ast.parse`/`bash -n` + `grep -cn`으로 기계적 판정(말로 착시 논쟁 말 것). 특히 send 줄 중복은 잡지 않았으면 실전 슬랙 2회 발송
- `[확정]` 편집마다 눈으로 확인(2번 "allow all edits"는 누르지 않음). 무인 시스템 수정의 안전장치
- `[확정]` SyntaxError 시 자동 `.bak` 복구 훅을 검사 명령에 내장

---

## G. 미해결 / 다음 작업

- `[TODO]` ★sunday/monday end-to-end 실제 LLM 실행 (`SKIP_SEND=1 MODE_OVERRIDE=sunday`(또는 monday) `./run_briefing.sh`) → 출력 품질 검증: 회고/전망 잘 나오나, 월요일 전망 가드가 추측 막나, 이한나가 주간 브리핑을 당일자로 반려 안 하나(로그 failed_criteria). 주의: AV weekly 8콜 + 한도(오늘 테스트로 소진 가능, 자정 리셋 후 권장) + LLM 2~4콜 비용. 머리 맑을 때 출력 읽고 판단 권장
- `[TODO]` 섹터 ETF(SMH/XLF/XLE/XLK 등)를 watchlist에 추가 = 주시종목 진짜 섹터별로 (별도 작업, market_fetch 종목 재설계)
- `[TODO]` 백업 `.bak` 파일 정리 정책
- `[TODO]` (CP15~16 이월) output 누적 정리, 봇 키 $50 한도 재점검(브리핑 LLM 같은 키·월 ~$2)·전용 키 분리 검토, 실패 알림(--notify-failure) 실작동 테스트, 멘션 겹침·송태섭 응답 안정성, 채소연 엔진, v2 6인 SOUL 파일화, 보안(VNC 암호·Tailscale 2FA·토큰 SecretRefs), 슬랙 owner·allowlist 유저 ID·리포명, 직보 예외(안감독·권준호) 증가 시 이한나 창구 무의미 방지 기준
- `[TODO]` (별도 프로젝트) CDA 엔진 WSL→맥미니 마이그레이션 + 권준호 TOOLS 호출부 채움 + showdown 채널 + Buksan Budongsan 앱

**크론/launchd 스케줄은 이번 작업에서 안 건드림. 현재 평일 파이프라인 안전 보존(weekday 폴백).**

# V2 "분리하되 이어주기" — 안전장치(schg) 적용

- **날짜**: 2026-06-23
- **선행**: CP17(주간모드 코드 완료) · 메모리 #13(V2 방향) · CP16(봇은 시스템 못 바꾼다 실증)
- **상태**: 안전장치 1차(로직 schg) 적용 완료 · 검증 통과 · 최종 검증(익일 06:00 자연 브리핑)은 관찰 대기
- **범위**: V2 착수 — 봇 권한 조사 → 안전장치 설계 → schg 적용

---

## A. 배경 — V2 목표와 출발점

**V2 "분리하되 이어주기"** = 슬랙에서 이한나(비서실장 봇)에게 말로 시키면 새벽 cron이 실제로 바뀌는 구조. 방식 B(설정파일 다리): 봇은 설정 데이터만 기록, cron은 돌 때마다 읽어 반영. 세 살림(cron 파이프라인 / OpenClaw 봇 / 엔진)은 분리 유지, config 파일 하나가 다리.

**[확정]** V2 성립의 선결 조건 = 봇이 파일을 쓸 수 있나. 이를 확인하러 OpenClaw 설정을 조사(읽기 전용).

---

## B. 봇 권한 조사 — 무제한 발견

**[확정] 봇 능력** (OpenClaw 설치본·설정 읽기로 확인)
- 두 봇(이한나=main 오케스트레이터, 송태섭=scout) 모두 `tools.profile: "coding"` (설정 최상위, 개별 override 없음)
- coding 프로파일이 켜는 도구: **파일쓰기**(write/edit/apply_patch) + **명령실행**(exec bash/process/code_execution) + 웹 + 메모리/세션 + cron + goal/plan/skill
- `readonly` 프로파일이 따로 존재하는데 coding을 선택 = 쓰기·실행 풀세트가 의도적으로 ON

**[확정] 권한/샌드박스**
- `sandbox.mode = "off"` (설정에 sandbox 없어 기본값) + `workspaceAccess = "none"`
- exec-filesystem-policy의 `isExecFilesystemConstrained`가 `sandboxMode !== "all"`이면 `return false` → 파일시스템 제약 비활성 → 봇이 워크스페이스 밖(`~/buksan-briefing` 등)도 자유롭게 read/write/exec
- exec 승인: `exec-approvals.json` 파일 없음 → 기본 `security="full"` + `ask="off"` = full bypass = 명령 검사·승인 모두 생략

**[확정] 결론**: 봇이 맥미니 전체에 무제한 권한, 승인 없음. `AGENTS.md`의 "Red Lines"는 행동 규범이지 코드 강제가 아님.

**[확정] CP16 미스터리 해소**: 봇이 "시스템 못 바꾼다"던 것은 능력 부족이 아니라 **설계 부재**였음. 능력은 처음부터 있었음.

---

## C. 결정적 사실 — 브리핑은 봇과 분리

**[확정]** 아침 브리핑은 봇이 아니라 시스템 crontab(`0 6 * * *` → run_briefing.sh → /usr/bin/python3)이 직접 돌림. 봇 exec 경로를 거치지 않음.
- crontab도 봇도 같은 사용자
- → 봇 권한을 잠가도 브리핑 무영향 (crontab은 로직 파일을 읽고 실행만, 쓰지 않음)
- 이 분리가 안전장치 설계의 전제: 봇을 조여도 브리핑이 안 깨짐

---

## D. 안전장치 설계 — 옵션 비교

검토 문서 `~/buksan-briefing/SECURITY_REVIEW.md`에 저장(읽기 전용 조사 + 제안).

**[확정] 핵심 정정**: exec 통제와 파일쓰기 통제는 별개 통제선.
- `exec-approvals.json`은 셸 명령(rm 등)만 막음
- 봇의 write/apply_patch 직접 파일쓰기는 exec 통제로 못 막음 → 파일 자체를 OS/커널 레벨로 막아야 함

**옵션 비교**

| 옵션 | 막는 것 | 부작용 | 판정 |
|---|---|---|---|
| A: exec-approvals allowlist | 셸 명령 | 브리핑이 셸 쓰면 위험·목록 의존 | 2순위 보강 |
| **B(채택): 로직만 schg** | **로직 손상·삭제 (write·exec 다)** | **거의 없음** | **★1순위** |
| C: sandbox.mode="all" | 워크스페이스 밖 전부 | docker 의존·재시작·브리핑 파손 | 제외 |
| (보류): config 잠금+sudoers | config 무검증 쓰기 | 같은 사용자 함정·복잡 | 보류 |

**[확정] 발상 전환 — 잠글 대상 뒤집기**: config(데이터)는 열고 로직(*.py/*.sh)만 불변화.
- 봇이 바꿔야 할 건 config뿐, 시스템 깨뜨리는 건 로직 손상
- config 깨져도 이미 cron이 실패 감지 + 슬랙 알림(market_fetch가 config 읽다 실패 → run_step 중단) = 기존 안전장치로 커버
- → update_config.py·sudoers·setuid 전부 불필요해짐

**[확정] "같은 사용자는 자기를 못 잠근다" 함정**: config를 잠그면 같은 사용자로 도는 갱신 스크립트도 막힘. 단순 chmod는 봇이 도로 +w. macOS는 스크립트 setuid 무시. root 소유+sudoers는 복잡 → config 잠금 방식(B案) 보류 근거.

---

## E. schg 적용 — 실행 및 검증

**[확정] 대상 분류** (적용 전 8개 파일 모두 플래그 없음 확인)
- **잠금(로직 5개)**: briefing_llm.py · market_fetch.py · news_fetch.py · send_briefing.py · run_briefing.sh
- **열어둠**: config.json(V2 대상) · output/ · *_spec.md · requirements.txt · .env
- **.bak 3개**: schg 안 거는 것 권장 (cron이 안 쓰니 무결성 무관, 걸면 백업 정리 시 noschg 번거로움)

**[확정] git 충돌 없음**: `~/buksan-briefing/`에 `.git` 없음(저장소 아님) → schg 충돌할 commit/checkout 없음.

**[확정/적용] 적용** (사용자가 Terminal에서 sudo 직접 — root 권한이라 에이전트 불가)
```
sudo chflags schg \
  <PROJECT>/briefing_llm.py \
  <PROJECT>/market_fetch.py \
  <PROJECT>/news_fetch.py \
  <PROJECT>/send_briefing.py \
  <PROJECT>/run_briefing.sh
```
- `ls -lO` 결과: 5개 모두 `schg` 플래그 확인
- config.json: schg 명령에서 제외 → 열린 상태 유지

**[확정/검증] 무부작용 검증 통과**
```
/usr/bin/python3 -B -m py_compile <4개 .py>   → PY OK
bash -n run_briefing.sh                        → SH OK
```
- schg 걸려도 읽기·파싱 정상 = 실행 체인 안 깨짐 = 브리핑 무영향 실증
- `-B`로 .pyc 캐시도 안 만듦

**[확정] schg 동작 원리**: 읽기·실행 허용, 쓰기·삭제·이름변경·chmod·chflags만 차단(커널 레벨 EPERM). 파일 단위 차단이라 상위 폴더가 봇 소유여도 삭제·이름변경까지 막음. 봇이 스스로 noschg 불가(root 전용) = 핵심 방어.

**[확정] 롤백**: `sudo chflags noschg <5개>` 한 줄, 즉시 복구. 재시작 불필요.

---

## F. 운영 규칙 변경

**[확정] 로직 수정 절차 변경**: buksan-briefing 로직 5개가 불변화됨. 봇도 사용자도(sudo 없이) 수정·삭제 불가. 향후 로직 수정 필요 시(CP17 후속 등) 반드시 3단계:
```
sudo chflags noschg <파일>   # 잠금 해제
# ... 수정 ...
sudo chflags schg <파일>     # 재잠금
```
config.json은 그대로 쓰기 가능.

**[추론] 적용 환경 주의**: schg 적용은 Terminal 앱 직접이 확실. Claude Code 세션 채팅창에 붙이면 텍스트로만 입력되고 실행 안 됨. `!` 접두는 sudo 비번 대화형 입력이 막힐 수 있음. (실제로 채팅창 붙여넣기로 1차 미적용 → Terminal에서 재실행 성공)

---

## 미해결 / 다음 작업

- **[TODO] ★최종 검증**: 익일(6/24) 06:00 자연 브리핑 정상 도착 확인(하루 관찰) = schg 실전 무해 최종 입증
- **[TODO] exec-approvals.json 추가**: 안전장치 2순위 보강(A). `security="allowlist"` + `ask="on-miss"` + `askFallback="deny"`. 허용목록 = 봇 90일 세션 로그의 실제 exec(cat/ls/which/diff/tail/env/printenv + openclaw status/gateway/dashboard 등). A는 셸만 통제, write/apply_patch는 schg가 막음 = 상호보완
- **[TODO] 하루 관찰**: 봇 대화·메모리 정상 확인
- **[TODO] ★V2 첫 기능 = mode_override**: "말로 시키면 바뀌는" 첫 입주자. config 읽기로 cron 안 건드리는 것부터. CP17 mode 구조 재활용
- **[TODO] 보틀넥(첫 기능 설계 시)**: 동시성(atomic write), 의도 해석(봇 변경 전 확인), 롤백(이력)
- **[TODO] cron 시각 변경**: 후순위. 봇 도구에 cron 있으나 내부 스케줄러인지 시스템 crontab 조작인지 미확인(나중 카드)
- **[TODO] CP17 마무리**: sunday/monday end-to-end 실제 LLM 실행(출력 품질 검증). ⚠️ 이제 로직 schg라 수정 필요 시 noschg 먼저

---

## 원칙 (이번 적용에서 적용)

- 위험 결정 정지: 시스템 권한 변경이라 머리 맑을 때 신중히
- measure-first: 봇 권한·샌드박스·exec 정책을 코드로 직접 확인 후 설계
- config 쓰기는 열되 로직 수정은 막기
- 평일 브리핑 무손상 최우선
- 안전장치가 본체를 안 깨뜨리는지 구조부터 검증(브리핑↔봇 분리 확인)

북산 비서실 — exec-approvals 화이트리스트 적용 기록
> 날짜: 2026-06-25
> 선행: V2 안전장치 1차(schg) 적용 완료, 6/24 무인 브리핑 정상 검증
> 상태: exec-approvals 적용 완료, 슬랙 승인 클라이언트 미작동 보류
> 결정 태그: [확정] 이사장 명시 / [승인] 제안→승인 / [추론] 맥락 기본값 / [TODO] 미해결
---
A. 사전 점검 (점검 1~3)
exec-approvals.json 적용 전, 봇의 실제 셸 사용 실태를 읽기 전용으로 조사했다.
A-1. 웹 도구의 셸 의존성 [확정]
web_search/web_fetch 구현 체인을 dist 끝까지 추적:
최종 네트워크 계층: `fetch-guard` → `globalThis.fetch` (Node 내장) + `undici` (Node 내장 HTTP)
child_process/exec/spawn/curl/wget: 전 구간 0건
결론: curl을 allowlist에서 빼도 봇 웹 기능 안 깨짐 [확정]
A-2. 봇 실제 exec 명령 목록화 [확정]
데이터 범위: 2026-06-20~21 (실 세션 로그 약 2일치, 시드의 "90일"은 보관 한도와 불일치)
봇	주요 바이너리	건수
main (한나)	grep 162, head 142, openclaw 77, cat 58, env 54, python3 54, echo 44, rm 34, which 20, tail 15, ls 14, notion 10, ntn 10, printenv 10	278
scout (송태섭)	cat 128, python3 112, ls 100, head 96, grep 80, echo 50, openclaw 48, tail 34, env 16, diff 16	292
초안 대비 누락: grep, head, echo, python3, notion/ntn → 추가 필요 [확정]
rm 34건: 전량 BOOTSTRAP.md 정리. allowlist 제외(ask 대상) [승인]
openclaw 서브명령: status/gateway/dashboard — 기존 목록 그대로 [확정]
A-3. curl/wget/nc 셸 사용 이력 [확정]
양쪽 봇 0건. python3 내부까지 전수 검사(urllib/requests/httpx/http.client) — 0건. 봇은 네트워크를 셸로 친 적 없음.
---
B. python3 보안 구멍 발견과 해결
B-1. 구멍 [확정]
python3를 allowlist에 넣으면 `python3 -c "import urllib...외부전송"` 한 줄로 .env 유출 가능. curl 차단이 무의미해짐.
B-2. 점검 4 — 패턴 매칭 의미론 확인 [확정]
OpenClaw dist에서 exec-approvals 매칭 로직을 끝까지 추적:
항목	결과
매칭 단위	세그먼트별 argv (파이프·연쇄 분해, 각각 검사)
실행파일 패턴	glob → 정규식 컴파일, basename/realpath 매칭
인자 패턴	argPattern 정규식 (argv[1:].join에 매칭)
셸 래퍼	sh -c / bash -c 내부 명령 추출 → 재귀 검사 (depth≤3)
서브셸/백틱	안전 분석 불가 시 analysis.ok=false → 승인 요청
인터프리터 인라인	strictInlineEval 내장 기능으로 별도 탐지
B-3. strictInlineEval [확정]
`tools.exec.strictInlineEval: true` 플래그:
python3 -c → 인라인 eval 탐지 → 헤드리스에서 거부
python3 script.py → 그대로 통과
대상 인터프리터: python/python2/python3/pypy/pypy3(-c), node/nodejs/bun/deno(-e/--eval), ruby(-e), perl(-e/-E), php(-r), lua(-e), osascript(-e)
doctor 감사에서도 인터프리터 allowlist 시 strictInlineEval 경고 출력
B-4. 봇 python3 호출 실측 [확정]
봇	python3 -c (인라인)	python3 file.py	비율
main	54건 (고유 1종)	0	100% 인라인
scout	112건 (고유 7종)	0	100% 인라인
전부 env 키 스캔 용도. strictInlineEval 적용 시 100% 승인 대상 — 이것이 정확히 차단 목표.
B-5. echo 안전성 [확정]
양쪽 리다이렉션(>) 0건, 서브셸 0건. 평문 출력만. allowlist에 넣어도 위험 낮음.
---
C. 화이트리스트 확정 및 적용
C-1. 최종 화이트리스트 [승인]
파일: `~/.openclaw/exec-approvals.json`
```json
{
  "version": 1,
  "defaults": {
    "security": "allowlist",
    "ask": "on-miss",
    "askFallback": "deny"
  },
  "tools": {
    "exec": { "strictInlineEval": true }
  },
  "agents": {
    "main": { "allowlist": [
      {"pattern":"cat"},{"pattern":"ls"},{"pattern":"which"},
      {"pattern":"diff"},{"pattern":"tail"},{"pattern":"head"},
      {"pattern":"grep"},{"pattern":"echo"},{"pattern":"env"},
      {"pattern":"printenv"},{"pattern":"python3"},
      {"pattern":"notion"},{"pattern":"ntn"},
      {"pattern":"openclaw"}
    ]},
    "scout": { "allowlist": [
      {"pattern":"cat"},{"pattern":"ls"},{"pattern":"which"},
      {"pattern":"diff"},{"pattern":"tail"},{"pattern":"head"},
      {"pattern":"grep"},{"pattern":"echo"},{"pattern":"env"},
      {"pattern":"printenv"},{"pattern":"python3"},
      {"pattern":"openclaw"}
    ]}
  }
}
```
C-2. 적용 검증 [확정]
JSON 파싱: `python3 -c "import json; json.load(...)"` 통과
파일 권한: 644, 865 bytes
agents 키: main/scout 2개 확인
strictInlineEval: true 존재 확인
재시작 불필요(매 exec마다 on-demand 읽기)
롤백: 파일 삭제 → full bypass 복귀
---
D. 승인자(approvers) 설정
D-1. approvers 위치 [확정]
exec-approvals.json이 아닌 `openclaw.json`의 `channels.slack.accounts.<계정>.execApprovals.approvers`에 위치.
형식: 슬랙 유저 ID 배열. 허용 표기: U***, user:U***, <@U***>.
D-2. ID 확인 [확정]
로그에서 발견된 U0BC2U5THM2 → 송태섭(봇) ID였음. 이사장 ID 아님
이사장 실제 ID: 슬랙에서 직접 확인하여 등록
교훈: 로그 ID를 검증 없이 적용하지 말 것
D-3. 적용 [확정]
openclaw.json의 default/scout 계정에 execApprovals.approvers 추가. 핫리로드 확인(gateway.log에서 config hot reload applied × 2).
---
E. 슬랙 승인 클라이언트 미작동 [TODO]
E-1. 증상
allowlist 밖 명령(rm) 시 승인 대기 타임아웃 → deny. 3회 시도 동일.
E-2. 원인 분석
`enabled` 기본값 = `"auto"` (approvers ≥ 1이면 true 평가) → 설정상 켜짐
게이트웨이 재시작(PID 변경 확인, clean shutdown → ready) 후에도 동일
네이티브 승인 클라이언트가 실제로 슬랙 DM을 발송한 흔적 0
로그에 승인 클라이언트 초기화 전용 로그가 없어 활성화 여부 불명
E-3. 판정 [승인]
슬랙 승인은 부가 기능. 핵심 목적(.env 키 유출 차단)은 `askFallback: deny`로 이미 달성. allowlist 밖 = 무조건 거부, 진짜 필요하면 Terminal 직접 실행. 추가 디버깅은 보류.
---
F. 현재 보호 상태 종합
계층	방식	상태
1층: 로직 불변	schg (5개 파일)	작동 중
2층: 셸 명령 제한	exec-approvals allowlist	작동 중
2층 보강: 인라인 eval 차단	strictInlineEval	작동 중
2층 보강: 키 유출 차단	curl/wget/nc 거부	작동 중
승인 경로	슬랙 네이티브	미작동 (보류)
---
G. 부수 확인: 날씨 API 브리핑 추가 가능성
data-go-kr-mcp 리포(공공데이터포털 MCP 서버) 확인. 기상청 단기예보 API를 MCP로 감싼 것.
이사장 목적은 "일일 브리핑에 날씨 한 줄 추가"
한나에게 MCP로 물리는 것(대화형)은 가능하나, 브리핑은 cron이 돌리므로 MCP 경로로는 안 닿음
정답: `weather_fetch.py` 추가해 cron 파이프라인에 직접 끼움 (market_fetch·news_fetch와 동일 구조)
공공데이터포털 가입 → 단기예보 활용신청 → 서비스키 발급 필요 [TODO]
---
미해결
항목	상태	비고
슬랙 승인 클라이언트	보류	핵심 목적 달성으로 우선순위 하락
openclaw status allowlist 매칭	미확인	allowlist에 있는데 막힌 원인 미규명
CP17 sunday/monday LLM 검증	미수행	schg 상태라 수정 시 noschg 먼저
V2 첫 기능 mode_override	다음 착수	config.json 읽기, CP17 mode 구조 재활용
날씨 브리핑 추가	검토	weather_fetch.py, 공공데이터포털 키 발급 필요
SCHD 종목 추가	미반영	V2 watchlist config로 해결 예정

