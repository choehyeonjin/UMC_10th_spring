미션 수행한 깃허브 브랜치: [feature/#8](https://github.com/choehyeonjin/UMC-10th-spring-study/commits/feature/%238)

- (필수) Spring Security를 적용하고 회원가입 API를 구현해주세요
  (폼 로그인을 위한 email, password를 추가로 받고 비밀번호는 BCrypt로 솔트처리해주세요)
    - 먼저 2주차에 구현한 회원가입 API 명세서와 Member 테이블 구조를 수정했다.
        - 폼 로그인을 위해 응답 본문에 email, password 필드를 추가했다.
        - password 컬럼을 추가했다.
        - social_uid와 social_type 컬럼을 nullable = true로 수정했다.
        - API 명세서
            - API Endpoint
                - `POST /auth/signup`
            - Request Body

                ```json
                {
                    "termIds" : [1,2,3],
                    "email": "test@test.com",
                    "password": "test",
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
                	"success": true,
                	"code": "MEMBER201_1",
                	"message": "회원가입에 성공했습니다.",
                	"data": {
                	  "createdAt": "2026-01-09T12:00:00"
                  }
                }
                ```

    - 동작 확인

      ![image.png](/week08/mission/mission_08_(1).png)

      ![image.png](/week08/mission/mission_08_(2).png)

      - 솔트 처리, 해싱된 password가 DB에 저장된 것을 확인할 수 있다.

- (필수) 회원가입 API는 Public API, 그 이외의 API는 Private API로 설정해주세요
  (Public API: 로그인 불필요 / Private API: 로그인 필요)
  (exceptionHandling을 구현해 인증, 인가 실패 시 응답이 통일되야 함)
    - 예외 핸들러와 SercurityConfig 구현
        - 인증, 인가 실패 예외 처리를 수행하고 응답 통일하는 커스텀 핸들러 `CustomEntryPoint`와 `CustomAccessDenied`를 구현했다.
            - 공통으로 사용되는 응답 통일 유틸 클래스는 HttpResponseUtil로 분리했다.

            ```java
            public class CustomEntryPoint implements AuthenticationEntryPoint {
            
                @Override
                public void commence(
                        HttpServletRequest request,
                        HttpServletResponse response,
                        AuthenticationException authException
                ) throws IOException {
            
                    HttpResponseUtil.setErrorResponse(response, GeneralErrorCode.UNAUTHORIZED);    }
            }
            
            public class CustomAccessDenied implements AccessDeniedHandler {
            
                @Override
                public void handle(
                        HttpServletRequest request,
                        HttpServletResponse response,
                        AccessDeniedException accessDeniedException
                ) throws IOException {
            
                    HttpResponseUtil.setErrorResponse(response, GeneralErrorCode.FORBIDDEN);    }
            }
            
            public class HttpResponseUtil {
            
                private static final ObjectMapper objectMapper = new ObjectMapper();
            
                public static void setErrorResponse(HttpServletResponse response, BaseErrorCode code) throws IOException {
                    // 응답 Content-Type, HTTP 상태코드 정의
                    response.setContentType("application/json;charset=UTF-8");
                    response.setStatus(code.getStatus().value());
            
                    // Response Body에 응답통일한 객체를 넣기
                    ApiResponse<Void> errorResponse = ApiResponse.onFailure(code, null);
            
                    // 실제 Response로 덮어쓰기
                    objectMapper.writeValue(response.getOutputStream(), errorResponse);
                }
            }
            ```

        - 두 핸들러를 빈으로 등록한 후 `.exceptionHandling`으로 SecurityConfig에 적용했다.
            - allowUris에 `"/api/v1/auth/**"`을 추가해 회원가입 API를 Public API로 등록했다.
    - 동작 확인

      ![image.png](/week08/mission/mission_08_(3).png)

      - Private API(마이페이지 조회)에 접근시 401 에러코드가 발생한다.