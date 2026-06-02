# Night Vision Web App

모바일 웹 브라우저에서 동작하는 야간 투시(Night Vision) 테스트 웹앱. 외부 빌드 도구 없이 단일 `index.html`로 구성되어 있습니다.

## 모드

- **필터 모드**: 카메라 영상에 실시간으로 밝기 부스트 + 야간 투시경 스타일의 초록색 톤 픽셀 변환 적용
- **장노출 모드**: 3초 동안 프레임을 누적(`max` 합산)하여 어두운 곳의 빛을 모은 정지 이미지를 모달로 표시 (저장 가능)
- **센서 모드**: WebXR `depth-sensing` 지원 여부를 탐지. 미지원 시 경고 메시지 표시

## 실행

HTTPS 환경에서만 카메라 권한이 동작합니다.

- GitHub Pages, Vercel, Netlify 등으로 배포하거나
- 로컬에서: `python -m http.server 8000` 후 `https://localhost` 환경(또는 mkcert로 인증서 생성) 사용
- 모바일 테스트는 `https://` 또는 `localhost`만 허용됨

## 권장 브라우저

- Android Chrome / Edge (WebXR depth-sensing 일부 지원)
- iOS Safari (카메라 / 필터 / 장노출 정상 동작, WebXR 미지원)

## 기술

- 순수 HTML / CSS / Vanilla JS
- `getUserMedia` (facingMode: environment)
- Canvas 2D pixel manipulation
- `Float32Array` 누적 버퍼 (장노출)
- WebXR Device API (`immersive-ar` + `depth-sensing`)
