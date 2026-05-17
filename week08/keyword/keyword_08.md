- Spring Security가 무엇인가?

  ## 1. Spring Security란?

  **Spring Security**는 Spring 기반 애플리케이션에서 **인증(Authentication)** 과 **인가(Authorization)** 를 담당하는 **보안 프레임워크**다.

  HTTP 요청이 컨트롤러에 도달하기 전에 **Security Filter Chain**을 통해 요청을 가로채고, 사용자가 누구인지, 어떤 권한을 가졌는지를 검증한다.
    
  ---

  ## 2. 동작 흐름

    ```
    Client 요청
      ↓
    Security Filter Chain
      ↓
    Authentication (인증)
      ↓
    SecurityContextHolder 저장
      ↓
    Authorization (인가)
      ↓
    Controller
    ```

    - 인증이 실패하면 → `AuthenticationEntryPoint`
    - 인가가 실패하면 → `AccessDeniedHandler`

    ---

  ## 3. 주요 구성 요소

  ### Security Filter Chain

    - Spring Security의 핵심
    - 여러 개의 Filter가 **체인 형태**로 연결되어 요청을 처리

  ### UserDetails / UserDetailsService

    - **UserDetails**: 인증된 사용자 정보 객체
    - **UserDetailsService**: 사용자 정보를 DB에서 조회하는 역할

  ### AuthenticationManager

    - 실제 인증 로직을 수행하는 매니저
    - 비밀번호 비교, 토큰 검증 등을 담당

  ### SecurityContext

    - 인증된 사용자 정보를 저장하는 컨텍스트
    - 세션 기반이면 세션에 저장
    - JWT 기반이면 요청마다 새로 생성

    ---

  ## 4. 장점

  ### 1. 강력한 보안 기능을 기본 제공

    - 비밀번호 암호화(BCrypt)
    - 세션 관리 자동 처리

  ### 2. 인증/인가 로직을 분리 가능

    - 비즈니스 코드와 보안 코드 분리
    - 컨트롤러/서비스 코드가 깔끔해짐

  ### 3. 다양한 인증 방식 지원

    - 세션 기반 로그인
    - JWT 기반 인증
    - OAuth2 / 소셜 로그인
    - 커스텀 인증 방식도 확장 가능

    ---

  ## 5. 단점

  ### 1. 러닝 커브가 높음

    - 필터 체인 구조 이해 필요
    - 설정 실수 시 원인 파악이 어려움

  ### 2. 초기 설정이 복잡

    - 기본 설정만 해도 로그인 페이지 자동 생성
    - REST API에서는 오히려 방해가 될 수 있음

  ### 3. 디버깅 난이도 높음

    - 컨트롤러까지 요청이 안 오고 필터에서 막힘
    - 에러 원인 파악이 쉽지 않음
- 인증(Authentication)vs 인가(Authorization)

  ## 1. 인증(Authentication) 흐름

  **인증은 사용자의 신원을 확인하는 과정**이다.

    1. **로그인 요청**
        - 사용자가 이메일·비밀번호 등의 인증 정보를 포함해 요청을 보낸다.
    2. **AuthenticationFilter**
        - `UsernamePasswordAuthenticationFilter`가 요청을 가로채

          `Authentication` 객체를 생성한다.

    3. **AuthenticationManager**
        - 인증 처리를 적절한 `AuthenticationProvider`에 위임한다.
    4. **AuthenticationProvider**
        - `UserDetailsService`를 통해 사용자 정보를 조회하고

          비밀번호를 검증한다.

    5. **SecurityContext 저장**
        - 인증에 성공하면 `Authentication` 객체가

          `SecurityContext`에 저장되어 이후 요청에서 사용된다.


    👉 **인증 성공 시: 사용자 정보가 SecurityContext에 저장됨**
    
    👉 **인증 실패 시: 요청 차단 (401 Unauthorized)**
    
    ---
    
    ## 2. 인가(Authorization) 흐름
    
    **인가는 인증된 사용자가 특정 리소스에 접근할 권한이 있는지 확인하는 과정**이다.
    
    1. **리소스 접근 요청**
        - 인증된 사용자가 보호된 리소스에 접근을 시도한다.
    2. **FilterSecurityInterceptor**
        - 요청을 가로채 권한 검사를 시작한다.
    3. **권한 비교**
        - `SecurityContext`에 저장된 사용자의 권한과
            
            리소스에 필요한 권한을 비교한다.
            
    4. **접근 결정**
        - 권한이 충분하면 접근을 허용한다.
        - 권한이 부족하면 접근을 거부한다.
    
    👉 **인가 실패 시: 403 Forbidden 발생**
    
    ---
    
    ## 3. 인증 vs 인가 비교
    
    | 구분 | 인증(Authentication) | 인가(Authorization) |
    | --- | --- | --- |
    | 질문 | 누구인가? | 할 수 있는가? |
    | 수행 시점 | 요청 초반 | 인증 이후 |
    | 기준 | 신원 정보 | Role |
    | 실패 응답 | 401 Unauthorized | 403 Forbidden |
    | 핵심 저장소 | SecurityContext | SecurityContext |
- Stateful vs Stateless

  ## Spring Security에서 Stateful vs Stateless

  Spring Security에서 **Stateful / Stateless**는 서버가 사용자의 로그인 상태를 기억하느냐에 대한 차이다.
    
  ---

  # 1. Stateful 방식

  ## 개념

  **서버가 사용자의 인증 상태를 저장하고 관리하는 방식**이다.

  대표적으로 **Session 기반 로그인**이 Stateful 방식이다.

  사용자가 로그인하면 서버는 로그인 정보를 **Session**에 저장하고, 클라이언트에게는 **JSESSIONID** 같은 세션 ID를 쿠키로 전달한다.

  이후 요청에서는 클라이언트가 세션 ID를 보내고, 서버는 세션 저장소에서 인증 정보를 찾아 사용자를 식별한다.
    
  ---

  ## 흐름

    ```
    1. 사용자가 로그인 요청
    2. 서버가 아이디/비밀번호 검증
    3. 인증 성공 시 서버에 Session 생성
    4. 클라이언트에게 JSESSIONID 쿠키 전달
    5. 이후 요청마다 JSESSIONID 전달
    6. 서버가 Session에서 사용자 인증 정보 조회
    ```
    
  ---

  ## 특징

    ```
    서버가 로그인 상태를 기억함
    인증 정보가 서버 세션에 저장됨
    클라이언트는 세션 ID만 보관함
    로그아웃 처리가 쉬움
    서버 확장 시 세션 공유 문제가 생길 수 있음
    ```
    
  ---

  ## 장점

    ```
    - 구현이 비교적 간단
    - 서버에서 로그인 상태를 완전히 제어 가능
    - 보안 정책 적용이 쉬움 (강제 로그아웃 등)
    ```
    
  ---

  ## 단점

    ```
    - 서버 메모리 사용
    - 서버 확장 시 세션 공유 문제 발생
    - 모바일/외부 API 서버에 부적합
    ```
    
  ---

  # 2. Stateless 방식

  ## 개념

  **서버가 사용자의 인증 상태를 저장하지 않는 방식**이다.

  대표적으로 **JWT 기반 인증**이 Stateless 방식이다.

  사용자가 로그인하면 서버는 사용자 정보를 담은 **토큰**을 발급한다.

  이후 클라이언트는 요청마다 토큰을 보내고, 서버는 토큰을 검증해서 사용자를 식별한다.

  서버는 세션을 저장하지 않는다.
    
  ---

  ## 흐름

    ```
    1. 사용자가 로그인 요청
    2. 서버가 아이디/비밀번호 검증
    3. 인증 성공 시 JWT 발급
    4. 클라이언트가 JWT 저장
    5. 이후 요청마다 Authorization 헤더에 JWT 전달
    6. 서버가 JWT 검증 후 사용자 인증 처리
    ```

  예시:

    ```
    Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
    ```
    
  ---

  ## 특징

    ```
    서버가 로그인 상태를 기억하지 않음
    인증 정보가 토큰에 담김
    클라이언트가 토큰을 저장하고 요청마다 보냄
    서버는 토큰 유효성만 검증함
    REST API 서버에 자주 사용됨
    ```
    
  ---

  ## 장점

    ```
    서버 확장이 쉽다
    세션 저장소가 필요 없다
    모바일 앱, SPA, REST API와 잘 맞는다
    서버 간 인증 상태 공유가 필요 없다
    ```
    
  ---

  ## 단점

    ```
    토큰 탈취 시 위험하다
    JWT는 한 번 발급되면 만료 전까지 무효화하기 어렵다
    로그아웃 처리가 Stateful보다 복잡하다
    토큰 크기가 세션 ID보다 크다
    Refresh Token 관리가 필요할 수 있다
    ```

  # 3. 세션 vs 토큰 비교

  | 구분 | 세션(Session) | 토큰(Token/JWT) |
      | --- | --- | --- |
  | 상태 | Stateful | Stateless |
  | 로그인 정보 | 서버 저장 | 클라이언트 보관 |
  | 확장성 | 낮음 | 높음 |
  | 서버 부하 | 세션 저장 필요 | 없음 |
  | 보안 제어 | 쉬움 | 상대적으로 어려움 |