# 북산 비서실 — web_search 활성화 기록

> 날짜: 2026-06-25
> 선행: V2 config 다리 구현 완료 (20번)
> 상태: web_search 활성화 완료, main/scout 양쪽 적용
> 결정 태그: [확정] 이사장 명시 / [승인] 제안→승인 / [추론] 맥락 기본값 / [TODO] 미해결

---

## A. 문제 발견 [확정]

슬랙에서 한나에게 "내일 월드컵 일정은?" → "web_search가 비활성이고 주요 사이트들도 동적 렌더링으로 내용 추출이 막히고 있습니다"

## B. 원인 분석 [확정]

### B-1. 3레이어 게이팅

| 레이어 | 조건 | 상태 |
|---|---|---|
| 프로파일 멤버십 | web_search가 coding 프로파일에 포함 | 통과 |
| provider surface | 검색 provider 설정 존재 | **미통과** (tools.web.search 블록 없음) |
| provider 선택 | allowKeylessAutoSelect | web_search: false / web_fetch: true |

### B-2. 핵심 코드 (dist/runtime-web-tools)

- web_search 호출부(line 727): `allowKeylessAutoSelect: false` → provider 미설정 시 DuckDuckGo조차 자동선택 안 됨
- web_fetch 호출부(line 823): `allowKeylessAutoSelect: true` → 무설정 자동 동작
- 결론: web_search는 `tools.web.search.provider` 명시 설정 필수 [확정]

### B-3. 설정 현황

- openclaw.json에 web_search/webSearch/search 관련 키 전무
- main/scout 둘 다 per-agent tools 오버라이드 없음 → 전역 coding 프로파일 상속
- 도구는 모델에 노출되나 provider 없어 실행 시 실질 작동 불가

---

## C. 해결 [확정]

### C-1. 설정 추가

openclaw.json의 tools 블록에 추가:

```json
"tools": {
  "profile": "coding",
  "web": {
    "search": {
      "provider": "duckduckgo"
    }
  }
}
```

- DuckDuckGo: 무인증(API 키 불필요). 유일한 키리스 provider [확정]
- 전역 설정: main(한나)과 scout(송태섭) 양쪽 동시 적용 [확정]

### C-2. 게이트웨이 재시작 필요 [확정]

핫리로드만으로는 provider 초기화 안 됨. `launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway`로 재시작 후 정상 작동.

패턴 재확인: openclaw.json에 새 기능 추가 시 핫리로드는 값만 갱신, 새 런타임 초기화는 재시작 필요. (exec-approvals approvers, 슬랙 승인 클라이언트에 이어 3번째)

---

## D. 검증 [확정]

슬랙에서 한나에게 월드컵 관련 질문 시리즈:

| 질문 | 결과 |
|---|---|
| 월드컵예선 32강을 위한 각조별 남은 경기 | 위키피디아에서 일정 확인, 조별 3라운드 날짜 정리 |
| 우리나라가 32강 진출할 수 있는 경우의 수 | A조 확인 → 순위표 정리 → 3위 진출 조건 계산 |

웹 검색 + 정보 정리 + 추론 모두 정상 작동 확인.

---

## E. 오늘(6/25) 전체 완료 요약

| 마일스톤 | 정리 번호 |
|---|---|
| exec-approvals 화이트리스트 적용 | 19번 |
| V2 config 다리 구현 (10개 항목) | 20번 |
| 슬랙 exec 승인 근본원인 확정 (Slack 네이티브 미구현) | 20번 |
| web_search 활성화 | 21번 (본 문서) |

---

## 미해결

| 항목 | 상태 | 비고 |
|---|---|---|
| 내일 아침 브리핑 관찰 | 대기 | SCHD 포함·뉴스 5건·monday once 반영 + once 소진 자동 삭제 확인 |
| weather_fetch.py | 미생성 | 공공데이터포털 서비스키 발급 선행 |
| 한나 능력 강화 | 미착수 | 메모·스케줄·리마인더 등 SOUL 설계 |
| V2 6인 SOUL 파일화 | 미착수 | 강백호·서태웅·정대만·채치수·안 감독·채소연 |
| 슬랙 exec 승인 | 보류 | 이 빌드에 Slack 네이티브 미구현 |

# 북산 비서실 — 남진모 SOUL 작성 + Ultron 구조 확정 기록

> 날짜: 2026-06-26
> 선행: V2 config 다리 구현 완료 (20번), web_search 활성화 (21번)
> 상태: 남진모 SOUL 6파일 완성, 슬랙 앱(Buksan Recon) 생성, 맥 배치 대기
> 결정 태그: [확정] 이사장 명시 / [승인] 제안→승인 / [추론] 맥락 기본값 / [TODO] 미해결

---

## A. config 다리 실전 검증 [확정]

아침 브리핑(2026-06-26 06:02 KST)에서 config 반영 3건 확인:

| 항목 | config 값 | 브리핑 반영 |
|---|---|---|
| SCHD 종목 추가 | watchlist에 SCHD | "SCHD +0.31%" 등장 |
| 뉴스 건수 | count_per_category: 5 | 카테고리당 5건 적용 |
| monday 모드 (once) | mode_override: {mode:"monday", type:"once"} | "§11 주간 브리핑 (월요일)" 적용 |

[TODO] once 소진 후 mode_override 자동 삭제 여부 — cat config.json으로 확인 필요

---

## B. 한나 월드컵 보고 — 내장 cron 발견 [확정]

### B-1. 내장 cron = OpenClaw 내부 스케줄러

한나에게 "아침 6시에 월드컵 결과 보내줘" → OpenClaw 내장 cron 도구로 자체 설정. 시스템 crontab이 아니라 내부 스케줄러 → exec-approvals와 무관하게 동작 [확정]

### B-2. 서브에이전트 발송 실패 → 메인 세션으로 수정 [확정]

첫 실행: 서브에이전트로 실행 → 슬랙 도구 미존재 + exec 차단 → 발송 실패
수정: "메인 세션에서 직접 보내" 지시 → 정상 작동

### B-3. 내장 cron vs 시스템 cron [승인]

| | 내장 cron | 시스템 cron |
|---|---|---|
| 설정 | 말로 시키면 끝 | 코드 작성 + schg |
| 안정성 | 게이트웨이 재시작 시 날아갈 수 있음 | 영구 |
| 용도 | 기간 한정, 가벼운 정기 보고 | 매일 빠지면 안 되는 것 |

---

## C. Ultron 폴더 구조 확정 [확정]

### C-1. 3구역 구조

| 구역 | 경로 | 원칙 |
|---|---|---|
| 1. 봇 | ~/.openclaw/workspace/ | OpenClaw 관리. 폴더명=영문 역할 id |
| 2. 엔진 | ~/projects/engines/ | 신규 엔진은 여기만. README.md = SSOT |
| 3. 기존 | 현재 위치 유지 | buksan-briefing, JT 등 안 옮김 |

### C-2. README.md 레지스트리 (engines/README.md)

| 엔진 | 포트 | 유형 | 경로 | 봇 id |
|---|---|---|---|---|
| cda | :8000 | B-1 상주 | ~/projects/engines/cda | recon |
| jt | 없음 | B-2 온디맨드 | ~/dev/osaka-blogger-discovery | traveler(미정) |
| sa | :8001 | B-1 (예정) | ~/projects/engines/sa | analyst(미정) |
| sra | 온디맨드 | B-2 (예정) | ~/projects/engines/sra | researcher(미정) |

### C-3. id 네이밍 변경 [확정]

- 남진모: analyst → **recon** (analyst는 SA 봇에 배정)
- 슬랙 앱명: Buksan Budongsan → **Buksan Recon**

---

## D. 보고 라인 재확정 [확정]

Ultron 전 방 공통 통보로 통일:

- 모든 봇 → 한나(Gatekeeper) 경유 → 이사장
- 한나 = 기준 검수만, 내용 무수정 전달 (LOOP 작업자↔채점자 분리)
- 통과 = 이사장에게 그대로 전달, 미달 = 봇에게 반려
- 직보 예외 = **안 감독 1인만**
- 기존 결정(남진모 직보) 변경됨 — 이사장 지시로 확인 [확정]
- 봇끼리 지시 안 함, 의견만 가능

---

## E. 남진모 SOUL 6파일 작성 [확정]

### E-1. 이름 변경

권준호 → 남진모 (슬램덩크 해남대부속고 감독, 지장형) [확정]

### E-2. 인물 카드

| 항목 | 값 |
|---|---|
| 이름 | 남진모 |
| 엔진 | 경제학자/상권전문가 |
| 표층 | 슬램덩크 해남대부속고 감독 |
| 시스템 id | recon |
| 슬랙 앱명 | Buksan Recon |
| 채널 | showdown |
| 보고 라인 | 한나 경유 (Gatekeeper, 무수정) |
| 유형 | B-1 (엔진 상주, 대화형 온디맨드) |

### E-3. 핵심 설계

- 존재목적: 권리금·보증금 날리지 않게 지키는 방패
- 톤: 직설. 모호한 낙관 금지
- 금지어: "대박날 것 같다" / "무난하다" / "잘 될 겁니다"
- 출력구조: ①한줄결론 ②입지치명타 ③데이터검증 ④실전액션플랜
- 내부 3관점: 현장 발품 → 데이터 분석 → 손익 계산
- 면책: 모든 분석 말미 고정
- Domain Knowledge: 흐르는/머무는 입지, 도깨비 상권 필터, 3중 배후지, 손익분기 월세 공식

### E-4. 6파일 목록

| 파일 | 내용 |
|---|---|
| SOUL.md | Core Identity, Personality, Speaking, Quotes, Emoji, Rules, Boundaries, Domain Knowledge |
| IDENTITY.md | id, 앱명, 소속, 역할, 보고라인, 채널, 유형 |
| USER.md | 이사장 정보, 기대사항, 대화방식 |
| AGENTS.md | 내부 3관점 (현장·데이터·손익) |
| TOOLS.md | CDA 엔진 연결부 [TODO: 엔진 확인 후 호출부 채움] + 웹 검색 |
| HEARTBEAT.md | 출력 표준 (4단계 구조 + 면책) |

배치 경로: `~/.openclaw/workspace/recon/`

---

## F. 슬랙 앱 생성 [확정]

Buksan Recon 앱 생성 완료:
- Socket Mode 활성화, App-Level Token(xapp-) 발급
- Bot Token Scopes: chat:write, app_mentions:read, channels:history, channels:read
- Event Subscriptions: app_mention, message.channels
- 워크스페이스 설치, Bot Token(xoxb-) 발급

---

## G. 기타 확인

### G-1. 슬랙 무료 플랜 [확정]

평가판 22일 후 자동 전환. 봇 운영에 영향 없음(앱 10개 한도, 현재 3개). 90일 메시지 기록 제한만 주의.

### G-2. 이미지 인식 권한 [TODO]

한나가 슬랙 이미지를 못 읽음. files:read 권한 추가 필요.
- 대상: Secretary, Scout, Recon 전부
- 추가 후 Reinstall to Workspace 필요

### G-3. 브리핑 검증 코멘트 — DELL 누락 [TODO]

monday 브리핑에서 본문에 DELL(-0.16%) 언급하면서 주시종목 표에서 누락. briefing_llm.py 프롬프트 개선 대상(schg 해제 필요, 급하지 않음).

---

## 미해결

| 항목 | 상태 | 비고 |
|---|---|---|
| 남진모 맥 배치 | 대기 | SOUL 6파일 배치 + openclaw.json 등록 + 게이트웨이 재시작 + 테스트 |
| CDA 엔진 마이그레이션 | 별도 세션 | 맥에서 돌아가는지 확인 → TOOLS.md 호출부 채움 |
| engines/README.md 생성 | 미착수 | 레지스트리 표 기록 (위험 0) |
| golden_standard 이동 | 미착수 | CDA 참조 실측 후 |
| files:read 권한 추가 | 미착수 | Secretary, Scout, Recon 전부 |
| once 소진 확인 | 미확인 | cat config.json으로 mode_override 삭제 여부 |
| 나머지 4인 SOUL | 미착수 | 안감독·강백호·서태웅·정대만·채치수 |
| weather_fetch.py | 미생성 | 공공데이터포털 서비스키 발급 선행 |

# 북산 비서실 — 디스패치 실증 + Slack→Discord 전환 결정 기록

> 날짜: 2026-06-29~30
> 선행: 남진모 SOUL 완성 + 슬랙 앱 생성 (22번)
> 상태: 내부 디스패치 작동 확인, Slack 봇↔봇 멘션 불가 확정, Discord 전환 결정
> 결정 태그: [확정] 이사장 명시 / [승인] 제안→승인 / [TODO] 미해결

---

## A. 남진모 맥미니 배치 [확정]

- SOUL 6파일: `~/.openclaw/workspace/recon/`에 배치 (6/6 확인)
- openclaw.json: agents.list + accounts.recon + bindings 등록, 토큰 교체
- 게이트웨이 재시작: 3계정(default/scout/recon) socket mode connected
- 슬랙 응답 확인: @Buksan Recon 정상 응답

---

## B. 디스패치 진단 + 해결 [확정]

### B-1. 진단 결과

- main→scout/recon 내부 디스패치: 역대 0건
- 원인: `subagents.allowAgents` 미설정 → 한나 자기 자신만 스폰 가능
- 브리핑 한나→태섭: 경유 안 함. cron이 main 직접 깨움
- subagent_runs 전수 확인: 4건 전부 자가 스폰

### B-2. 해결

- main per-agent에 `allowAgents: ["scout", "recon"]` 추가 [확정]
- agents.defaults가 아닌 main per-agent에 넣은 이유: defaults에 넣으면 scout/recon에도 상속돼 봇끼리 디스패치 열림 (한나만 오케스트레이터)
- 게이트웨이 재시작 후 한나→진모/태섭 sessions_spawn 호출 → 양쪽 응답 확인
- ★Ultron 6단계 ⑥배선 검증 첫 실증 [확정]

---

## C. 봇↔봇 슬랙 멘션 — 3회 실패, 플랫폼 제한 확정 [확정]

### C-1. 3회 실측

| 시도 | 조치 | 결과 |
|---|---|---|
| 1차 | 한나 @Buksan Recon 멘션 | 무응답. 이벤트 미도착 |
| 2차 | allowBots: "mentions" + users: [3봇 ID] + botLoopProtection | 무응답. 설정 로드 확인, 이벤트 미도착 |
| 3차 | 3봇 message.groups 이벤트 구독 추가 + Reinstall | 무응답. 이벤트 미도착 |

### C-2. 근본 원인

- gateway.log + 상세로그: 한나 멘션 시점 scout/recon에 app_mention/message 이벤트 0건
- OpenClaw 드롭이 아님. Slack이 봇 발신 멘션을 이벤트로 보내지 않음
- bot_id가 찍힌 메시지는 다른 앱에 app_mention 미발화 [확정]

### C-3. 진단 과정에서 확인된 사항

| 항목 | 확인 내용 |
|---|---|
| replyToMode | 봇 간 수신과 무관 (아웃바운드 스레드 설정) |
| allowBots | 진짜 설정이지만, 이벤트 자체가 안 와서 무력 |
| daily-briefing 채널 | 비공개(private). groups:read 스코프 필요 |
| 3봇 채널 멤버 | 전원 입장 확인 |
| Bot User ID | main=U0BBU6JFYR3, scout=U0BC2U5THM2, recon=U0BEDNS5MCG |

---

## D. Slack→Discord 전환 결정 [확정]

### D-1. Discord가 해결하는 이유

| 항목 | Slack | Discord |
|---|---|---|
| 봇 메시지 이벤트 | app_mention 미발생 (플랫폼 강제) | MESSAGE_CREATE 전체 발생 (봇 포함) |
| 봇 메시지 무시 | 플랫폼 강제 차단 | 개발자 선택 (코드 컨벤션) |
| OpenClaw 지원 | 있음 | 있음 + 네이티브 exec 승인 존재 |

### D-2. 확정 사항

- 목표: 슬랙 무료 전환(~7/22) 전 20일 내 Discord 전환
- 병행 운용(B안) 확정: 7일 병행, 슬랙 폴백 유지
- 채널 구조: 단일 채널 시작 → 운용 후 확장
- 봇 등록: 3봇 먼저 → 나머지 6봇은 배치 시 Discord 직접 신설
- 시한: 22일 목표, 봇↔봇 대화 테스트 통과가 진짜 마일스톤. 늦어지면 완화 OK

### D-3. 마이그레이션 4단계

| 단계 | 기간 | 내용 |
|---|---|---|
| 1 | 2~3일 | Discord 서버 + 3봇 앱 생성 (이사장 직접) |
| 2 | 5일 | openclaw.json 재배선 + send_briefing.py 수정 + 봇↔봇 대화 테스트 |
| 3 | 7일 | 슬랙·Discord 병행 운용 |
| 4 | 이후 | 슬랙 종료 + 미배치 6봇 Discord 신설 |

### D-4. send_briefing.py 병행 발송 구조 [승인]

config 다리에 `send.platform` 필드 추가: "slack" → "both" → "discord" 단계 전환. 기존 슬랙 코드 유지, schg 해제→수정→검증→schg 필요.

---

## E. SOUL 6종 수신 [확정]

Ultron에서 도착:

| 봇 | 파일 | 유형 |
|---|---|---|
| 서태웅 | SEOTAEWOONG_SOUL.md | B-2 (SA) |
| 강백호 | KANGBAEKHO_SOUL.md | B-2 (JT) |
| 권준호 | KWONJUNHO_SOUL.md | B-2 (WB) |
| 정대만 | JEONGDAEMAN_SOUL.md | B-2 (MB) |
| 채소연 | CHAESOYEON_SOUL.md | C형 윤활제 (가설) |
| 채치수 | CHAECHISU_SOUL.md | C형 회의론자 |

9봇 중 8봇 SOUL 완성. 안 감독만 골격.

---

## F. 기타 완료 항목

- files:read: 3봇 전부 추가 완료 [확정]
- groups:read: Recon 추가 + Reinstall [확정] (Secretary/Scout는 이미 보유)
- 이사장 슬랙 ID(U0BBR70F883): 한나 MEMORY에 등록 [확정]
- 한나 MEMORY 봇 호출 규칙: 최종 갱신 대기 (Discord 전환 후)

---

## G. 아침 브리핑 분석 (6/29 monday 모드)

- config 다리 반영: SCHD 포함, monday 모드 적용 [확정]
- 반복 문제: 주시종목 표에서 DELL·SCHD 누락 (2회 연속)
- 포맷5칸 미달 (2회 연속)
- 관전포인트 3번 항목이 빈 면책 ("확인 불가, 별도 확인 권장")
- briefing_llm.py 프롬프트 보강 필요 [TODO] (schg 해제 필요)

---

## H. 매몰비용 인정

Discord 전환으로 폐기되는 작업:
- allowBots: "mentions" 설정
- users 허용목록 등록
- botLoopProtection 설정
- groups:read 스코프 추가 + Reinstall
- message.groups 이벤트 구독 추가

가치: Slack 플랫폼 제한이라는 진짜 원인을 드러내 준 진단 과정. 헛걸음이 아니라 실측.

---

## 미해결

| 항목 | 상태 | 비고 |
|---|---|---|
| Discord 마이그레이션 단계 1 | 다음 착수 | Discord 서버 + 3봇 앱 생성 |
| 남진모 CDA 엔진 연결 | CDA 방 | 봇 가동 중, 배선만 |
| 봉투 사이클 첫 실증 | CDA 연결 후 | 남진모 CDA→봉투→한나 검수→이사장 |
| 안 감독 SOUL 정식 작성 | 미착수 | 유일 미완 |
| briefing_llm.py 프롬프트 보강 | 미착수 | DELL 누락 + 포맷5칸 반복 문제 |
| 한나 MEMORY 봇 호출 규칙 최종 갱신 | Discord 전환 후 | 슬랙 의존 표현 정리와 함께 |
| send_briefing.py 병행 발송 구조 | 단계 2에서 | config send.platform 추가 |
