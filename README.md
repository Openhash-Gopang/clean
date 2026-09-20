# clean.hondi.net — K-Cleaner (구 fiil.kr)

혼디(Hondi) 생태계의 환경 신고·AI 자동 분석 서비스(K-Cleaner, SP-14)의 웹앱입니다.
기존 `fiil.kr`에서 이 저장소(`clean.hondi.net`)로 이전되었습니다.

## 이전하면서 함께 고친 문제

혼디 메인 AI 비서(`gwp-registry.js` → `_gwpLaunch()`)는 이 서비스를 새 탭으로 열 때
URL 쿼리 파라미터로 다음을 이미 전달하고 있었습니다.

- `token` — 사용자 GUID(익명 식별자)
- `facts` (base64 JSON) — 혼디 비서가 이미 확보한 부가 정보(`currentLocation` 포함)
- `ctx` (base64) — 사용자의 원래 발화

그런데 옛 `fiil.kr` 화면은 이 파라미터들을 전혀 소비하지 않고 있었습니다. 그 결과:

1. **더미 사용자 정보** — 실제 로그인 사용자 대신 "김제주"라는 하드코딩된 값이 표시됨.
2. **GPS 시간 초과** — 혼디가 이미 위치를 계산해 넘겨줬는데도, 이 페이지가 다른 오리진에서
   자체적으로 `navigator.geolocation`을 다시 호출하면서 권한/응답 지연으로 타임아웃.
3. (부수 발견) 이 페이지 자체에 별도 LLM API 키를 관리자 대시보드에서 설정해야
   AI 분석이 동작하는 구조 — 혼디의 다른 K-서비스들과 달리 플랫폼 공용 백엔드를
   쓰지 않고 있었음.

`webapp.html`은 이를 다음과 같이 고칩니다.

- `token`으로 `GET https://hondi-proxy.tensor-city.workers.dev/profile?guid=<token>`을
  호출해 실제 이름/연락처를 표시합니다(더미 데이터 제거).
- 위치는 **① 첨부 사진의 EXIF GPS → ② 혼디가 넘겨준 `facts.currentLocation` →
  ③ 이 페이지의 브라우저 GPS(최후 수단, 타임아웃 8초로 명확화)** 순으로 사용합니다.
- AI 사진 분석은 혼디 공용 프록시(`POST /deepseek`, `deepseek-chat` 비전 모델)를
  사용합니다. 이 저장소 자체에 별도 API 키를 설정할 필요가 없습니다.
- 신고 접수가 끝나면 `GWP_DONE` postMessage로 opener(혼디 메인 탭)에 결과를
  돌려보내, 다른 K-서비스와 동일하게 PDV(개인 데이터 볼트)에 기록되도록 합니다.

## 함께 반영해야 하는 hondi 저장소 쪽 변경

`Openhash-Gopang/hondi`에도 아래 변경이 필요합니다(별도 커밋/PR로 준비함).

- `gwp-registry.js` — `fiil-kcleaner.url` → `https://clean.hondi.net/webapp.html`
- `src/gopang/gwp/allowed-origins.js` — `GWP_ALLOWED_ORIGINS`에서 `fiil.kr` → `clean.hondi.net`
- `worker.js` — CORS `ALLOWED_ORIGINS`에서 `fiil.kr` → `clean.hondi.net`
- `services/fiil-kcleaner/manifest.json` — `url` 갱신
- `prompts/SP-14_kcleaner_v1.4.txt` — archive에만 있던 최신 SP를 최상위로 승격
  (`prompts/sp-catalog.json`이 `SP-14_kcleaner` 키를 찾지 못해 `sp_url`이 계속
  `null`이 되던 결함 수정)
- `desktop.html`, `pages/k-services.html`, `pages/agents.html` — 사용자에게 노출되는
  `fiil.kr` 바로가기 링크를 `clean.hondi.net`으로 갱신

## 배포 전 확인이 필요한 부분

- `/deepseek` 요청 바디 스키마(OpenAI 호환 `messages` 배열 + `image_url`)는
  `worker.js`의 `callDeepSeek()` 코드를 근거로 구성했습니다. 실제 배포 전
  한 번 실물 응답으로 검증해주세요.
- Supabase 신고 이력 저장(`_updateFiilReport`)은 `hondi` 저장소 쪽에 이미
  "PocketBase 목적지 미확정"으로 비활성 처리되어 있어(TODO, `state.js` 참고)
  이 웹앱에서도 별도로 구현하지 않았습니다. 신고 결과는 `GWP_DONE`으로
  혼디 PDV에는 기록되지만, 별도의 신고 이력 DB가 필요하면 그 부분을
  먼저 결정해야 합니다.
