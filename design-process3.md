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
