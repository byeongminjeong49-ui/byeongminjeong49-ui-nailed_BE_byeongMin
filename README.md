<div align="center">

# Nailed — 인증(로그인/로그아웃) & 마이페이지

중고 의류·IT기기 거래 플랫폼 프로젝트에서 팀장을 맡아 **Nailed**에서 로그인/로그아웃, JWT 보안, 비밀번호 찾기, 마이페이지를 설계·구현했습니다.

👤 담당 역할 (정병민)

**로그인/로그아웃 · JWT 인증 · 비밀번호 찾기** 도메인을 백엔드부터 프론트엔드까지 단독으로 설계·구현했습니다. 팀원이 각자 자신의 파트를 구현한 공용 화면인 **마이페이지**에서는 프로필·정산계좌 조회/수정 파트를 담당했습니다.

### 담당 도메인 한눈에 보기

| 영역 | Backend | Frontend |
|---|---|---|
| 인증 | `AuthController`, `AuthServiceImpl`, `JwtTokenProvider` | `authApi.js`, `AuthPages.jsx` |
| 마이페이지 *(담당 파트)* | `MemberController`, `MemberService` | `UserProfilePage.jsx`, `myPageApi.js` |

### 파일별 구현 내용

- `AuthController` — 회원가입/로그인/로그아웃/토큰 재발급/비밀번호 재설정 API
- `AuthServiceImpl` — 로그인 실패 카운트 및 계정 잠금, 임시 비밀번호 발급, 관리자 사칭 방지 키워드 필터링
- `JwtTokenProvider` — Access/Refresh Token 발급·검증, `type` 클레임으로 토큰 오용 차단
- `authApi.js` — 토큰 저장, 만료 임박 시 자동 재발급, 401 응답 처리하는 axios 인터셉터
- `MemberController` / `MemberService` — 마이페이지 프로필·정산계좌 조회/수정 API

### 핵심 기술 성과: 토큰 이중 저장 & 자동 재발급

XSS로부터 Refresh Token을 보호하기 위해 저장 위치를 이원화하고, 토큰 만료로 인한 로그아웃 경험을 최소화했습니다.

- Access Token → `sessionStorage`, Refresh Token → `HttpOnly` 쿠키
- 만료 10초 전 `/api/auth/refresh` 자동 호출 후 원 요청 1회 재시도
- 재발급까지 실패하면 로컬 세션 삭제 후 로그인 페이지로 강제 이동
- Access/Refresh 모두 `type` 클레임을 넣어 토큰을 바꿔 쓰는 오용을 서버에서 차단

### 로그인 실패 제한 (Brute Force 방지)

```
30분 내 5회 연속 로그인 실패 → 10분 계정 잠금
```

- 실패 카운트를 예외 발생 중에도 커밋해야 해서 `@Transactional(noRollbackFor = CustomException.class)` 적용
- `ACTIVE / LOCKED / SUSPEND / BANNED / WITHDRAWN` 상태별로 로그인 차단 사유 분리

### 비밀번호 찾기 & 마이페이지

- `SecureRandom` 기반 10자리 임시 비밀번호 발급(혼동 문자 제외) 후 `BCrypt` 저장
- 마이페이지 정산계좌의 예금주명은 BE에서 회원 실명으로 강제 설정해 FE 임의 변경 차단
- 모든 마이페이지 요청은 토큰에서 추출한 회원 ID 기준으로만 조회
