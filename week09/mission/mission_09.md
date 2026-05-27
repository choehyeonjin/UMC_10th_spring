- 미션 수행한 깃허브 브랜치: [feature/#9](https://github.com/choehyeonjin/UMC-10th-spring-study/tree/feature/%239)
- JWT + OAuth 구현 (Spring OAuth-Client 이용)
    - 기존 폼 로그인(이메일+비밀번호) 방식을 제거하고, 소셜 로그인 전용 방식으로 개편했다.
        1. 소셜 로그인 요청: 브라우저에서 `GET /oauth/authorize/{provider}` (예: kakao)로 접속하여 소셜 인증을 시작한다.
        2. 콜백 및 분기 처리: 카카오 인증이 완료되면 Spring Security(`OAuthSuccessHandler`)에서 DB를 조회하여 가입 여부를 판단한다.
            - 기존 회원 (로그인): 즉시 JWT(Access Token)를 JSON 데이터로 발급한다.
            - 신규 회원 (회원가입): JWT를 발급하지 않고, 추가 정보 입력이 필요함(`REQUIRE_SIGNUP`)을 알리고, 카카오에서 획득한 소셜 정보(socialUid, email 등)를 JSON으로 반환한다.
        3. 최종 회원가입: 신규 회원이 응답받은 소셜 정보와 추가 정보(약관, 성별 등)를 포함하여 **`POST /api/v1/auth/signup`** API를 호출하면, 최종적으로 DB에 저장되고 JWT가 발급된다.
    - 이에 따라 기존 폼 로그인 방식에서 사용하던 `password` 필드는 엔티티와 DTO에서 모두 제거하고, 유저를 식별하기 위한 `socialType` (소셜 제공자)과 `socialUid` (소셜 고유 식별자) 필드를 추가하여 로직을 수정했다.
    - 소셜 로그인 인증 성공 시, 프론트엔드는 status 값으로(REQUIRE_SIGNUP, LOGIN_SUCCESS) 신규 또는 기존 회원임을 판단하고 회원가입 또는 홈 화면을 호출하게 된다.

        ```java
        // 2. 분기 처리 후 JSON 조립
                if (member.getId() != null) {
                    // 기존 회원 -> 로그인 성공 처리 및 JWT 발급
                    response.setStatus(HttpStatus.OK.value());
                    String accessToken = jwtUtil.createAccessToken(new AuthMember(member));
        
                    // 데이터 조립
                    Map<String, Object> data = new HashMap<>();
                    data.put("status", "LOGIN_SUCCESS");
                    data.put("accessToken", accessToken);
        
                    BaseSuccessCode successCode = MemberSuccessCode.LOGIN_OK;
                    ApiResponse<Map<String, Object>> responseBody = ApiResponse.onSuccess(successCode, data);
        
                    objectMapper.writeValue(response.getOutputStream(), responseBody);
        
                } else {
                    // 신규 회원 -> 회원가입이 필요함을 알리고 소셜 정보를 JSON으로 반환
                    response.setStatus(HttpStatus.ACCEPTED.value());
        
                    Map<String, Object> data = new HashMap<>();
                    data.put("status", "REQUIRE_SIGNUP");
                    data.put("socialType", member.getSocialType().toString());
                    data.put("socialUid", member.getSocialUid());
                    data.put("email", member.getEmail());
                    data.put("name", member.getName());
        
                    BaseSuccessCode successCode = MemberSuccessCode.SOCIAL_AUTH_OK;
                    ApiResponse<Map<String, Object>> responseBody = ApiResponse.onSuccess(successCode, data);
        
                    objectMapper.writeValue(response.getOutputStream(), responseBody);
                }
        ```

- 회원가입 API 명세서
    - API 개요
        - 소셜 인증을 마친 신규 유저(미가입자)가 추가 정보를 입력하여 최종적으로 회원가입을 완료하고 JWT를 발급받는 API입니다.
    - API Endpoint
        - `POST /auth/signup`
    - Request Body

        ```json
        {
        	"socialType": "KAKAO",
          "socialUid": "3456789012",
        	"termIds" : [1,2,3],
          "name": "최현진",
          "gender": "FEMALE",
          "birthdate": "2004-08-31",
          "address": "서울시 노원구 월계동",
          "preferredFoodIds": [1, 2, 4]
        }
        ```

    - Request Header

        ```json
        Content-Type: application/json
        ```

    - Response
        - 201 Created

        ```json
        {
          "isSuccess": true,
          "code": "MEMBER201_1",
          "message": "회원가입에 성공했습니다.",
          "result": {
            "memberId": 2,
            "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
          }
        }
        ```

- 테스트

  ![image.png](week09/mission/mission_09_(1).png)

    - 먼저 http://localhost:8080/oauth/authorize/kakao에 접속해 소셜 로그인(신규 회원)을 수행한다.

  ![image.png](week09/mission/mission_09_(2).png)

    - socialUid, socialType과 추가 정보를 입력하여 회원가입을 진행하고 액세스 토큰이 발급된다.

  ![image.png](week09/mission/mission_09_(3).png)

    - 스웨거에서 액세스 토큰을 입력해 authorize 수행시 마이페이지가 성공적으로 조회된다.

  ![image.png](week09/mission/mission_09_(4).png)

    - DB에 해당 회원이 저장된다.

  ![image.png](week09/mission/mission_09_(5).png)

    - http://localhost:8080/oauth/authorize/kakao 에 다시 접속시, 바로 액세스 토큰이 발급되고 같은 회원으로 로그인에 성공한다.
- 한계점과 차이점
    - 현재 구현한 Spring OAuth2-Client 방식은 스프링 시큐리티가 전체 인증 흐름을 제어하는 방식이다.
        - 동작 원리: 사용자가 카카오 로그인을 완료하면, 카카오 서버는 프론트엔드를 거치지 않고 백엔드 콜백 주소로 인가 코드(code)를 직접 전달한다. 백엔드는 내부적으로 토큰 교환 및 유저 정보를 조회한 후, 최종 결과(JSON 또는 리다이렉트)를 프론트엔드로 반환한다.
        - 장점: `SecurityConfig` 설정만으로 구현이 가능하여 백엔드 코드가 간결해진다. 프론트엔드의 처리 로직이 거의 없어 초기 개발 및 협업 속도가 빠르다.
        - 단점: 프론트엔드와 백엔드의 결합도가 높다. 브라우저 리다이렉트에 강하게 의존하므로, 추후 모바일 앱(iOS, Android) 클라이언트 확장 시 구조적 제약이 발생한다.
    - REST API 방식 (프론트엔드 주도형)은 프론트엔드와 백엔드의 완전한 분리를 위해 최근 실무에서 주로 채택하는 아키텍처다.
        - 동작 원리:
            1. 프론트엔드에서 카카오 로그인 페이지를 호출한다.
            2. 인증 완료 후, 카카오 서버가 프론트엔드 콜백 주소( `localhost:3000/oauth/callback`)로 인가 코드(code)와 상태 값(state)을 반환한다.
            3. 프론트엔드가 URL에서 해당 파라미터를 파싱하여 백엔드 REST API(`GET /auth/oauth2/kakao/callback?code=xxx`)로 전송한다.
            4. 백엔드는 수신한 인가 코드를 이용해 카카오 서버와 통신하여 유저 정보를 획득하고 최종적으로 JWT를 발급한다.
        - 장점: 프론트엔드와 백엔드가 완벽하게 분리된다. 백엔드는 인가 코드를 수신하여 JWT를 반환하는 순수 API 서버로 동작하므로, 웹뿐만 아니라 모바일 앱 등 다중 클라이언트 환경에 유연하게 대응할 수 있다.
        - 단점: URL 파싱, API 호출 처리 등 프론트엔드 측의 구현 복잡도와 작업량이 증가한다.