- 빌더패턴이란?
    - 자바에서 빌더 패턴은 **객체를 단계적으로 생성**할 수 있게 해주는 디자인 패턴이다.
      **생성자에 파라미터가 많을 때** 가독성과 안정성을 높이기 위해 사용한다.
    - 사용 이유
        - 일반 생성자 방식

            ```java
            Member m = new Member("홍길동", 25, "010-1234-5678", "SEOUL");
            ```

            - 어떤 값이 어떤 필드인지 가독성 낮음
        - 빌더 방식

            ```java
            Member m = Member.builder()
                    .name("홍길동")
                    .age(25)
                    .phone("010-1234-5678")
                    .address("SEOUL")
                    .build();
            ```

            - 순서 상관 없음
            - 필드명으로 값이 어떤 의미인지 명확함
            - 선택적 파라미터 처리 용이 (null 허용 필드 생략 가능)
            - 실제로는 내부 static 클래스가 자동 생성됨

            ```java
            public static class MemberBuilder {
                private String name;
                private int age;
            
                public MemberBuilder name(String name) { this.name = name; return this; }
                public MemberBuilder age(int age) { this.age = age; return this; }
            
                public Member build() {
                    return new Member(name, age);
                }
            }
            ```

- record vs static class
    - public static DTO
        - 사용하는 상황
            - 요청 DTO
            - 필드가 많고 선택적 파라미터가 많은 대형 DTO (옵션/기본값/중첩 구조 등…)
            - Service에서 여러 도메인 데이터 합쳐 Response로 만드는 경우
            - 테스트 코드에서 필드를 다양하게 조합할 때
        - 예시

            ```java
            public class TestResDTO {
            
                @Builder
                @Getter
                public static class Testing {
                    private String testing;
                }
            }
            ```

    - record
        - 사용하는 상황
            - 단순 요청 DTO/응답 DTO
                - 요청: 프론트엔드 → 백엔드로 전달되는 JSON 바디를 그대로 받는 객체
            - 필드 수가 적고, 빌더까지 필요 없음
            - 조회용 api
                - 쿼리 결과를 그대로 받는 용도
                - Setter나 빌더 필요 없음
        - 예시

            ```java
            public record TestTesting(String testing) { }
            ```

        - 표 비교

      | 항목 | inner static DTO (lombok 조합) | record DTO |
              | --- | --- | --- |
      | 의미 | 한 도메인의 여러 응답/요청 스키마를 한 파일/네임스페이스로 묶음 | 순수 데이터 캐리어(불변)를 간결하게 표현 |
      | 파일 구조 | `TestResDTO.Testing`, `MemberResDTO.Summary`처럼 한 클래스 안에 public static으로 다수 정의 → 파일 수 ↓ | 보통 레코드별 파일 1개 → 파일 수 ↑ (패키지로 그룹핑) |
      | 가변성 | Lombok 조합에 따라 가변/불변 선택 가능 (`@Getter` + no `@Setter` = 사실상 불변) | 기본 불변 (모든 컴포넌트 final 성격) |
      | 빌더 지원 | `@Builder` 로 바로 지원 → 선택 파라미터/대형 DTO에 유리 | 기본 미제공. 필요하면 별도 빌더 구현 or 다른 패턴 사용 |
      | 기본값 처리 | `@Builder.Default`로 매우 간단 | 생성자에서 직접 지정해야 함 |
      | 가독성 | “도메인별/응답별 묶음”을 한 파일에서 한눈에 보기 좋음
        다만 파일 커짐→ PR 충돌 가능 | 파일은 늘어나지만 각 DTO가 독립 → 충돌↓, 이식/재사용↑ |
      | 요청 vs 응답 | 응답: `@Builder`로 생성 가독성↑
        요청: 빌더 불필요 | 요청/응답 모두 가능하나 응답(불변)에 더 자연스러움 |
- 제네릭이란?

  타입을 일반화해서 재사용성을 높이는 문법이다. → 클래스나 메서드를 만들 때 타입을 나중에 결정

    1. 기본 문법

    ```java
    class Box<T> {
        private T value;
    
        public void set(T value) {
            this.value = value;
        }
    
        public T get() {
            return value;
        }
    }
    
    Box<String> box = new Box<>();
    box.set("hello");
    
    String str = box.get();
    ```

  2. 타입 파라미터

       자주 쓰는 이름:

        - T : Type
        - E : Element
        - K : Key
        - V : Value

        ```java
        class Pair<K, V> {
            private K key;
            private V value;
        }
        ```

    2. 제너릭 메서드

        ```java
        public <T> void print(T value) {
            System.out.println(value);
        }
        ```

- @RestControllerAdvice이란?
    - 역할: 컨트롤러 전역에서 발생하는 예외를 공통 포맷 JSON으로 응답
    - 구성: 클래스 레벨에 `@RestControllerAdvice`, 메서드 레벨에 `@ExceptionHandler`
    - 장점:
        - 하나의 클래스로 모든 컨트롤러에 대해 전역적으로 예외 처리가 가능함
        - 중복 제거(try-catch문 삭제) → 가독성 및 유지보수성 향상
        - 직접 정의한 에러 응답을 일관성있게 클라이언트에게 내려줄 수 있음
    - 구성:
        1. 도메인 서비스에서 도메인 예외(`TestException`) 발생

        ```java
        // domain: test
        @Override
        public void checkFlag(Long flag){
            if (flag == 1){
                throw new TestException(TestErrorCode.TEST_EXCEPTION);
            }
        ```

        1. `TestException`은 `GeneralException`를 상속

        ```java
        // global
        @Getter
        @AllArgsConstructor
        public class GeneralException extends RuntimeException {
            private final BaseErrorCode code;
        }
        
        // domain
        public class TestException extends GeneralException {
            public TestException(BaseErrorCode code) { super(code); }
        }
        ```

        1. 전역 핸들러가 `GeneralException`을 한 번에 처리
        - `TestException` 핸들러가 있으면 그게 먼저, 없으면 `GeneralException`, 그래도 없으면 `Exception`

        ```java
        @RestControllerAdvice
        public class GeneralExceptionAdvice {
        
            @ExceptionHandler(GeneralException.class)
            public ResponseEntity<ApiResponse<Void>> handleException(GeneralException ex) {
                return ResponseEntity.status(ex.getCode().getStatus())
                        .body(ApiResponse.onFailure(ex.getCode(), null));
            }
        
            @ExceptionHandler(Exception.class)
            public ResponseEntity<ApiResponse<String>> handleException(Exception ex) {
                BaseErrorCode code = GeneralErrorCode.INTERNAL_SERVER_ERROR;
                return ResponseEntity.status(code.getStatus())
                        .body(ApiResponse.onFailure(code, ex.getMessage()));
            }
        }
        ```

        1. 컨트롤러는 성공 로직만 작성

        ```java
        @RestController
        @RequiredArgsConstructor
        @RequestMapping("/temp")
        public class TestController {
        
            private final TestQueryService testQueryService;
        
            @GetMapping("/exception")
            public ApiResponse<TestResDTO.Exception> exception(@RequestParam Long flag) {
                testQueryService.checkFlag(flag); // 예외 처리
                return ApiResponse.onSuccess(GeneralSuccessCode.OK,
                        TestConverter.toExceptionDTO("This is Test!"));
            }
        }
        ```

- Optional이란?
    - 값이 있을 수도 있고 없을 수도 있는 상황을 명시적으로 표현하기 위한 클래스이다.
    1. 사용 이유

       기존 방식:

        ```java
        User user = findUser();
        
        if (user != null) {
            System.out.println(user.getName());
        }
        ```

       문제:

        - null 체크를 계속 해야 함
        - 실수로 빠뜨리면 NullPointerException 발생

    2. Optional 개념

    ```java
    Optional<User> user = findUser();
    ```

  의미:

    - 값이 있음 → Optional 내부에 존재
    - 값이 없음 → 비어 있음 (empty)

    3. 생성 방법

    ```java
    Optional.of(user);         // null 허용 안됨
    Optional.ofNullable(user); // null 허용
    Optional.empty();          // 빈 Optional
    ```

    4. 사용 위치

    - 반환값으로 사용

        ```java
        Optional<User>findUserById(Long id);
        
        User user = userRepository.findById(id)
            .orElseThrow(() -> new NotFoundException());
        ```

        - 값이 없을 수 있다는 것을 명확하게 표현