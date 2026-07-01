<div align="center">

# Nailed — 인증(로그인/로그아웃) & 마이페이지

중고 의류·IT기기 거래 플랫폼 **Nailed**에서 로그인/로그아웃, JWT 보안, 비밀번호 찾기, 마이페이지를 설계·구현했습니다.

![Java](https://img.shields.io/badge/Java-21-orange?logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3-6DB33F?logo=springboot&logoColor=white)
![React](https://img.shields.io/badge/React-Vite-61DAFB?logo=react&logoColor=black)
![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1?logo=mysql&logoColor=white)

</div>

<br/>

## 담당 파일

| 영역 | 파일 |
|---|---|
| Backend | `AuthController` `AuthServiceImpl` `MemberController` `MemberService` `JwtTokenProvider` `JwtAuthenticationFilter` `SecurityConfig` |
| Frontend | `authApi.js` `myPageApi.js` `AuthPages.jsx` `UserProfilePage.jsx` |

<br/>

## 로그인 / 로그아웃 — 핵심 설계

| 항목 | 구현 내용 |
|---|---|
| 토큰 분리 저장 | Access Token → `sessionStorage`(탭 종료 시 삭제), Refresh Token → `HttpOnly` 쿠키. JS로 접근 불가능한 쿠키에 Refresh Token을 둬서 XSS 공격 시에도 탈취 불가 |
| 쿠키 자동 전송 | Axios `withCredentials: true` + BE `SecurityConfig`의 `allowCredentials(true)`를 쌍으로 맞춰 CORS 환경에서도 쿠키 정상 전송 |
| Access Token 자동 재발급 | 만료 10초 전부터 `POST /api/auth/refresh` 자동 호출 → 새 토큰으로 원래 요청 1회 재시도 (`axios` 인터셉터 구조) |
| 재발급 실패 처리 | Refresh Token까지 만료/무효면 로컬 세션 전체 삭제 후 로그인 페이지로 강제 이동 (`clearAuthStorage({ redirect: true })`) |
| 로그아웃 | BE: DB에 저장된 `refresh_token` 삭제 + 쿠키 만료(`maxAge(0)`) 응답 → 서버 측에서도 완전 무효화<br/>FE: `try/finally`로 API 실패와 무관하게 로컬 세션은 항상 정리되도록 보장 |
| JWT 타입 클레임 구분 | Access/Refresh 토큰 모두 `type: "access" \| "refresh"` 클레임 포함 → 재발급 API에 Access Token을 잘못 넣거나 그 반대인 경우를 서버에서 검증하여 차단 |
| 로그인 실패 제한 (Brute Force 방지) | 30분 내 5회 연속 로그인 실패 시 10분 계정 잠금. 실패 카운트를 예외 발생 중에도 DB에 커밋해야 해서 `@Transactional(noRollbackFor = CustomException.class)` 적용 |
| 계정 상태별 로그인 차단 | `ACTIVE / LOCKED / SUSPEND / BANNED / WITHDRAWN` 상태를 분리 관리해 각 상태별로 다른 에러 메시지 반환 |

<br/>

## 로그인 요청 흐름

```
1. POST /api/auth/login (userid, password)
2. 계정 상태 확인 (잠금/정지/탈퇴 여부)
3. BCrypt로 비밀번호 검증
   실패 시 → 실패 카운트 증가, 5회 누적 시 10분 잠금 후 예외 반환
   성공 시 → 실패 카운트 초기화, 로그인 횟수/최근 로그인 시각 갱신
4. Access Token(1시간) + Refresh Token(7일) 발급
5. Refresh Token → HttpOnly 쿠키로 응답 헤더에 실어 전달, DB에도 저장
6. Access Token → 응답 바디로 반환 → FE가 sessionStorage에 저장
```

<br/>

## 비밀번호 찾기

| 항목 | 구현 내용 |
|---|---|
| 임시 비밀번호 생성 | `SecureRandom` 기반 10자리 (혼동 문자 `0 O 1 I l` 제외) |
| 저장 | 생성 즉시 `BCrypt`로 해시하여 DB 반영 |

<br/>

## 마이페이지

| 항목 | 구현 내용 |
|---|---|
| 프로필 조회/수정 | 닉네임, 샵 소개, 프로필 이미지 관리 (이미지 삭제 시 DB `NULL` + 기본 이미지 노출) |
| 주문/정산 | 구매/판매 주문 내역, 정산 내역·계좌 조회. 예금주명은 BE에서 회원 실명으로 강제 설정하여 FE에서 임의 변경 불가 (금융 사고 방지) |
| 회원 탈퇴 | 실제 삭제 대신 `member_status = 'WITHDRAWN'` 소프트 삭제로 거래·리뷰 데이터 보존 |
| 인가 처리 | 모든 요청은 `SecurityUtil.getCurrentMemberId()`로 토큰에서 추출한 회원 ID 기준으로만 조회 → 타인 정보 접근 원천 차단 |

<br/>

## API 엔드포인트

| Method | URL | 설명 |
|:---:|---|---|
| `POST` | `/api/auth/signup` | 회원가입 |
| `POST` | `/api/auth/login` | 로그인 (Access Token 응답 + Refresh Token 쿠키 발급) |
| `POST` | `/api/auth/refresh` | Access Token 재발급 |
| `POST` | `/api/auth/logout` | 로그아웃 (DB 토큰 삭제 + 쿠키 만료) |
| `POST` | `/api/auth/password/reset-request` | 임시 비밀번호 발급 |
| `GET` `PUT` | `/api/members/mypage/profile` | 프로필 조회/수정 |
| `GET` | `/api/members/mypage/orders` | 주문 내역 |
| `GET` `PUT` | `/api/members/mypage/account-info` | 정산 계좌 |
| `DELETE` | `/api/members/mypage` | 회원 탈퇴 |

<br/>

## 트러블슈팅

| 문제 | 원인 | 해결 |
|---|---|---|
| Refresh Token 쿠키 미전송 | CORS `allowCredentials` ↔ FE `withCredentials` 설정 불일치 | 양쪽 설정 일치시켜 해결 |
| 회원가입 시 저장 안 됨 | 서비스 클래스 레벨 `@Transactional(readOnly = true)`가 쓰기 메서드까지 적용 | 쓰기 메서드에 `@Transactional` 재선언 |
| 로그인 실패 카운트 미저장 | 실패 시 예외 발생으로 트랜잭션 롤백되어 카운트 미반영 | `noRollbackFor` 옵션 적용 |


