web/auth/controller/AuthController.java       # 회원가입·로그인·로그아웃·토큰 재발급·비밀번호 재설정
web/auth/service/AuthServiceImpl.java         # 인증 핵심 로직
web/auth/dto/AuthRequest.java, AuthResponse.java

web/member/controller/MemberController.java   # 마이페이지 API
web/member/service/MemberService.java
web/member/entity/Member.java
web/member/repository/MemberRepository.java

config/jwt/JwtTokenProvider.java              # JWT 발급·검증
config/jwt/JwtAuthenticationFilter.java       # 요청별 토큰 검증 필터
config/SecurityConfig.java                    # CORS, 인증 정책
