# NovelAI · Gemini · 캐릭터 챗 – Codex 가이드

## 1) 프로젝트 개요
- Express 서버(`proxy-server.js`)가 `public` 정적 파일을 서빙하며 NovelAI 배치 생성, Gemini 챗/썸네일, 캐릭터 롤플레잉 챗 생성기를 제공한다.
- 프론트는 HTML/CSS/Vanilla JS. 주요 번들:
  - NovelAI 배치 생성: `batch-generator.html|css|js`
  - Gemini: `gemini.html|css|js`
  - 캐릭터 챗 생성기: `index.html` → `start.html|style_start.css|js_start.js` → `Main_chat.html|Main_chat_style.css|dark_mode.css` + 로직 `main.js`, `Api.js`, `config.js`, `chat.js`, `message.js`, `storage.js`, `summary.js`, `ui.js`.
- 권장 Node 20.x(최소 18). 스크립트: `npm start`(prod), `npm run dev`(nodemon). 의존성: `express`, `cors`, `@google/genai`.

## 2) 실행
1. `npm install`
2. `npm start` 또는 `npm run dev`
3. 브라우저에서 `http://localhost:3000` (루트는 `batch-generator.html`, `/gemini.html`은 Gemini, 캐릭터 챗은 파일 경로로 접근하거나 정적 루트에서 직접 연다)

## 3) 상태/스토리지 키
- NovelAI: `state` ↔ `localStorage` `novelai_batch_state`.
- Gemini: `google_api_key`, `gemini_chat_model`, `gemini_chat_rooms`, `gemini_active_chat_room`.
- 캐릭터 챗: `characterFormData`(캐릭터 작성 폼), 세션 저장소 키 `gemini_chat_sessions`(config.STORAGE_KEY), 요약 레벨 `summary_level`, Gemini API 키 `gemini_api_key`(Api.js).

## 4) NovelAI 배치 생성 (`public/batch-generator.*`)
- 모드: `continuous` / `memoPad`(`updateModeUI` 토글).
- 상태: API 키, 태그, 캐릭터 최대 3명, 메모장 최대 50개, 이미지 크기·샘플링·CFG, `requestDelay`, `varietyPlus`, 레퍼런스 이미지(+style-aware,strength), `generatedImages`.
- 흐름: `startContinuousGeneration`/`startMemoPadGeneration` → `validateSettings` → `/api/novelai/generate-image` → ZIP 파싱(JSZip) → `state.generatedImages` → 미리보기/다운로드. 로딩/진행도 UI 유지.
- 프롬프트: v4.5 `v4_prompt`/`v4_negative_prompt`; `varietyPlus` 시 `skip_cfg_above_sigma`; 레퍼런스는 `director_reference_*`에 base64(URI-encoded).
- 다운로드: 단일 a[href], 전체 JSZip.

## 5) Gemini (`public/gemini.*`)
- 모드: 썸네일/챗/합성/테스트. 포트 5500/5501에서 API_BASE_URL을 :3000으로 교체.
- 모델: Gemini 3 Pro(기본) / Gemini 2.5 Pro. v1beta REST, `thinkingConfig` 미전송.
- 챗: `/api/gemini/chat` 프록시 호출. 로컬 채팅방(최대 5개 페이지네이션, 자동 생성, 삭제, 저장)과 모델 배지, 이미지 첨부 지원. 새로고침 시 빈 새 방을 기본 활성화하지만 기존 방·기록은 로컬스토리지에 남음.
- 썸네일/합성: `/api/gemini/image`, base64 이미지+프롬프트, `responseModalities:["TEXT","IMAGE"]`, safety OFF. 합성은 배경+캐릭터 배열 처리, 재시도 로직 포함.

## 6) 캐릭터 롤플레잉 챗 생성기 (public/index.html 등)
- 흐름: `index.html`(랜딩) → `start.html`에서 캐릭터/프롬프트 입력(`js_start.js`가 `characterFormData` 저장) → `Main_chat.html`에서 대화.
- 주요 모듈:
  - `main.js`: 앱 상태/이벤트, 메시지 편집·재생성, 요약 모달 트리거, Gemini 스트리밍 호출 연동.
  - `Api.js`: `@google/generative-ai` ESM 사용, `MODEL_NAME`(config, 기본 gemini-2.5-pro)으로 `sendToGemini`/`sendToGeminiStream`, 토큰/비용 로깅 포함.
  - `config.js`: 모델명, 스토리지 키, 요약 시스템 프롬프트 생성기, 요약 레벨 기본값.
  - `chat.js`: 시스템 프롬프트 템플릿 구성, 세션 저장/불러오기/삭제, 채팅 리스트 렌더용 데이터 구성.
  - `message.js`: 메시지 수정/삭제/재롤 등의 핸들러.
  - `storage.js`: localStorage CRUD, 세션 ID 생성.
  - `summary.js`: 요약 모달, 요약 실행(`sendToGemini` 호출), 요약 레벨 관리(`summary_level`), 결과를 상태/저장소에 반영.
  - `ui.js`: DOM 캐시, 메시지 렌더/상태 표시, 모달 open/close, 기본 이벤트 도우미.
- 스타일: `Main_chat_style.css`, `dark_mode.css`(다크 토글), `style_start.css`.

## 7) 백엔드 API (`proxy-server.js`)
- 정적 서빙: `/` → `batch-generator.html`, `/gemini.html` → Gemini. `public` 전체 노출.
- `POST /api/novelai/generate-image`: NovelAI 프록시. `director_reference_images`는 `decodeURIComponent` 후 전달. 성공 시 ZIP 바이너리 그대로 반환.
- `POST /api/gemini/chat`: 기본 모델 `gemini-3-pro-preview`, safety OFF, temp 0.8/topK 40/topP 0.95. 이미지 인라인 파트+history 지원. `thinkingConfig` 미전송.
- `POST /api/gemini/image`: 모델 `gemini-3-pro-image-preview`, 프롬프트 보강, `responseModalities:["TEXT","IMAGE"]`, safety OFF. 첫 이미지 파트 base64 반환, 없으면 에러 메시지.

## 8) 체크리스트
1. NovelAI 연속/메모 모드 각 1회 생성: 레퍼런스/varietyPlus/CFG 반영, ZIP 파싱·다운로드 확인.
2. Gemini: 챗/썸네일/합성 각 1회, 이미지 첨부/미첨부, 채팅방 생성·전환·삭제·페이지네이션·새로고침 후 복귀 확인.
3. 캐릭터 챗: `start.html`→`Main_chat.html` 플로우, 세션 저장/로드/삭제, 요약 모달 동작, 다크모드 토글 확인.
4. 슬라이더/로딩/진행도 UI 및 한국어 텍스트 인코딩 정상 여부 확인.
5. 코드 스타일: ES2015+, 템플릿 리터럴/화살표 함수, 코멘트 최소. CSS는 기존 패턴 재사용.
6. 보안/로깅: API 키·민감 프롬프트 로그 금지. 백엔드 에러 로그는 상태코드·요약만.

## 9) 참고/주의
- Gemini 2.5 Pro: v1beta REST에서 `thinkingConfig` 불가(미전송). 모델명은 사용 환경에 맞게 조정 필요.
- 캐릭터 챗 요약 프롬프트는 `config.js`에 하드코딩(한국어 요약 템플릿, ALLOW_OMISSION=OFF).
- 브라우저 다운로드만 지원. 서버 저장/메타데이터는 별도 API 필요.
- 다중 이미지/대화 세션이 많을 때 로컬스토리지/메모리 사용량 주의.
