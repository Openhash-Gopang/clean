# K-Clean (clean.hondi.net)

제주 스마트 신고 서비스 "K-Clean"의 혼디(hondi.net) 통합 버전입니다.
원본은 `nounweb/fiil` (구 `fiil.kr`)이며, 이 저장소는 그 실제 소스를
그대로 옮기고 아래 결함을 수정한 것입니다.

## 이번에 옮기며 고친 결함

1. **로그인 사용자 대신 더미가 표시됨** — `webapp.html`의 신고 화면 /
   마이페이지 / 결과 화면 세 곳에 "김제주 · 010-****-5678"이 정적으로
   하드코딩되어 있었고, 이를 실제 값으로 대체하는 코드가 전혀 없었습니다.
   혼디가 GWP로 넘겨주는 `token`(혼디 계정 guid)으로
   `hondi-proxy.tensor-city.workers.dev/profile?guid=`를 조회해 실제
   이름/전화로 교체하도록 `_loadHondiProfile()` / `_applyProfileToUI()`를
   추가했습니다. 신고 접수 시 Supabase에 저장되는 `reporter` 필드도
   더 이상 GUID를 전화번호인 것처럼 저장하지 않습니다.
2. **GPS "시간 초과"** — 혼디 비서는 실제로 위치 정보를 매번 정상적으로
   전달하고 있었지만, 계약이 어긋나 있었습니다. 혼디(`engine.js`)는
   신형 계약(`facts`/`facts_enc`, `currentLocation`)만 보내고 있었는데,
   K-Clean의 `gwp-sdk.js`/`initGPS()`는 구형 계약인 평문 `gps_addr`
   파라미터만 읽고 있어 매번 브라우저 GPS를 새로 요청하다 타임아웃
   났습니다. 혼디 쪽에 `gps_addr`를 하위호환으로 추가 전송하도록
   고쳐 즉시 반영되도록 했습니다 (별도 hondi 저장소 패치).
3. **컨텍스트(ctx) 디코딩 불일치** — 혼디는 `ctx`를 base64로 인코딩해
   `ctx_enc=b64`로 표시해 보내는데, `gwp-sdk.js`는 항상 plain
   `decodeURIComponent`만 수행해 한글 컨텍스트가 깨질 수 있었습니다.
   `ctx_enc=b64`일 때 base64 디코딩하도록 `gwp-sdk.js`를 수정했습니다.
4. **"관리자 대시보드에서 LLM API 키를 설정하세요" 배너가 항상 표시됨**
   — 이 배너는 사용되지 않는 `fiil_llm_config` 키만 확인했는데, 실제
   1차 분석 경로(`_openaiFetch` → drone-proxy Worker → 서버측 Gemini
   키)는 클라이언트 키가 필요 없어 정상 동작 중에도 매번 오탐 경고가
   떴습니다. 해당 설정은 DeepSeek 폴백 전용이므로, 폴백이 실제로
   실패할 때만 알리도록 배너를 제거하고 콘솔 로그로 대체했습니다.
5. **GUID를 전화번호로 오용** — `onInit()`에서 `localStorage.setItem
   ('gopang_phone', token)`처럼 익명 토큰을 전화번호 키에 저장하던
   부분을 제거하고, 토큰은 `gopang_token`에, 표시용 전화번호는 실제
   프로필 조회 결과에서만 가져오도록 분리했습니다.

## 배포

- GitHub Pages(Deploy from branch: `main` / root) + Cloudflare DNS
  (`clean` → `openhash-gopang.github.io`, DNS only)로 서비스됩니다.
- `CNAME` 파일이 `clean.hondi.net`으로 설정되어 있습니다(원본은
  `fiil.kr`).
- `dashboard.html`(관리자 대시보드)과 별도 Cloudflare Worker인
  "drone-proxy"(`worker.js`, `wrangler.toml`, Gemini Vision 분석 전담)는
  기존과 동일하게 별도로 운영됩니다 — 이 Worker는 `git push`로
  자동배포되지 않으며 Cloudflare 대시보드/`wrangler deploy`로 수동
  배포해야 합니다.
