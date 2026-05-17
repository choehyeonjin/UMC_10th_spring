- Page와 Slice

  `Page`와 `Slice`는 Spring Data JPA에서 **페이징 조회 결과를 담는 타입**이다.

  ## Page

  `Page`는 **전체 데이터 개수까지 함께 조회하는 페이징 결과**다.

    ```java
    Page<Member> members = memberRepository.findAll(pageable);
    ```

  예를 들어 전체 회원이 100명이고, 한 페이지에 10명씩 보여준다면 `Page`는 다음 정보를 알 수 있다.

    ```
    members.getContent();// 현재 페이지 데이터
    members.getTotalElements();// 전체 데이터 개수
    members.getTotalPages();// 전체 페이지 수
    members.getNumber();// 현재 페이지 번호
    members.hasNext();// 다음 페이지 존재 여부
    ```

  즉, `Page`는 **1페이지, 2페이지, 3페이지처럼 전체 페이지 수가 필요한 화면에 적합하다.**

  단점은 전체 개수를 구하기 위해 **count query**가 추가로 실행될 수 있다는 점이다.
    
  ---

  ## Slice

  `Slice`는 **전체 데이터 개수는 조회하지 않고, 다음 페이지가 있는지만 확인하는 페이징 결과**다.

    ```
    Slice<Member>members=memberRepository.findAll(pageable);
    ```

  `Slice`는 다음 정보 정도를 알 수 있다.

    ```
    members.getContent();// 현재 페이지 데이터
    members.getNumber();// 현재 페이지 번호
    members.hasNext();// 다음 페이지 존재 여부
    ```

  하지만 전체 개수나 전체 페이지 수는 모른다.

  대신 **count query**를 하지 않기 때문에 `Page`보다 성능상 유리할 수 있다.

  보통 요청한 개수보다 1개 더 조회해서 다음 페이지 존재 여부를 판단한다.

  예를 들어 size가 10이면 11개를 조회한다. 11개가 있으면 다음 페이지가 있다고 판단하고, 실제 응답에는 10개만 내려준다.
    
  ---

  ## 언제 Page/Slice를 쓰는가

  `Page`는 **전체 페이지 수가 필요한 경우**에 사용한다.

  예를 들어 게시판 하단에 페이지 번호가 필요하면 `Page`가 적합하다.

  이 경우 전체 게시글 수와 전체 페이지 수를 알아야 하기 때문이다.

  `Slice`는 **다음 페이지가 있는지만 알면 되는 경우**에 사용한다.

  예를 들어 무한 스크롤이나 더보기 버튼 방식에서는 전체 페이지 수가 필요 없다.

  이 경우 현재 데이터와 `hasNext`만 있으면 된다.

- Java stream API

  `Java Stream API`는 **컬렉션 데이터를 반복문 없이 함수형 스타일로 처리할 수 있게 해주는 기능**이다.

  ## Stream이란?

  `Stream`은 데이터를 담는 저장소가 아니다. `List`, `Set`, `Array` 같은 데이터 원본을 흘려보내면서 필터링, 변환, 정렬, 집계 등을 처리하는 흐름이다.

    ```
    원본 데이터 → stream → 중간 연산 → 최종 연산 → 결과
    ```
    
  ---

  ## Stream의 기본 구조

  Stream은 보통 3단계로 사용한다.

    ```
    1. stream 생성
    2. 중간 연산
    3. 최종 연산
    ```
    
  ---

  ## 1. stream 생성

  컬렉션에서 stream을 만들 수 있다.

    ```java
    names.stream()
    ```

  배열에서도 만들 수 있다.

    ```java
    Arrays.stream(arr)
    ```
    
  ---

  ## 2. 중간 연산

  중간 연산은 데이터를 가공하는 과정이다.

  대표적으로 `filter`, `map`, `sorted`, `distinct`가 있다.

  ### filter

  조건에 맞는 데이터만 남긴다.

    ```java
    list.stream()
            .filter(n -> n > 10)
            .toList();
    ```

  `n > 10`인 값만 남긴다.
    
  ---

  ### map

  데이터를 다른 형태로 변환한다.

    ```java
    list.stream()
            .map(n -> n * 2)
            .toList();
    ```

  각 값을 2배로 바꾼다.

  DTO 변환할 때도 많이 쓴다.

    ```java
    List<MemberResponse> responses = members.stream()
            .map(member -> new MemberResponse(member.getName(), member.getAge()))
            .toList();
    ```
    
  ---

  ### sorted

  정렬한다.

    ```java
    list.stream()
            .sorted()
            .toList();
    ```

  객체 정렬은 이렇게 한다.

    ```java
    members.stream()
            .sorted(Comparator.comparing(Member::getAge))
            .toList();
    ```

  나이 기준 오름차순 정렬이다.
    
  ---

  ### distinct

  중복을 제거한다.

    ```java
    list.stream()
            .distinct()
            .toList();
    ```
    
  ---

  ## 3. 최종 연산

  최종 연산은 Stream 처리를 끝내고 결과를 만든다.

  대표적으로 `toList`, `forEach`, `count`, `findFirst`, `anyMatch`, `collect`가 있다.

  ### toList

  결과를 리스트로 만든다.

    ```java
    List<Integer> result = list.stream()
            .filter(n -> n > 10)
            .toList();
    ```
    
  ---

  ### forEach

  각 요소를 하나씩 실행한다.

    ```java
    list.stream()
            .forEach(System.out::println);
    ```

  단순 출력이나 처리에 사용한다.
    
  ---

  ### count

  개수를 센다.

    ```java
    long count = list.stream()
            .filter(n -> n > 10)
            .count();
    ```
    
  ---

  ### findFirst

  첫 번째 값을 가져온다.

    ```java
    Optional<Member> member = members.stream()
            .filter(m -> m.getName().equals("kim"))
            .findFirst();
    ```

  결과가 없을 수도 있어서 `Optional`로 반환된다.
    
  ---

  ### anyMatch

  조건을 만족하는 값이 하나라도 있는지 확인한다.

    ```java
    boolean exists = members.stream()
            .anyMatch(m -> m.getAge() >= 20);
    ```
    
  ---

  ### collect

  결과를 원하는 자료구조로 모은다.

    ```java
    List<String> result = names.stream()
            .collect(Collectors.toList());
    ```

  Java 16 이후에는 간단히 `toList()`를 많이 쓴다.
    
  ---

  ## Stream 특징

  Stream은 원본 데이터를 바꾸지 않는다.

    ```java
    List<Integer> numbers = List.of(3, 1, 2);
    
    List<Integer> sorted = numbers.stream()
            .sorted()
            .toList();
    ```

  이때 `sorted`는 정렬된 리스트지만, 원래 `numbers`는 그대로다.
    
  ---

  Stream은 최종 연산이 있어야 실행된다.

    ```java
    names.stream()
            .filter(name -> name.startsWith("k"));
    ```

  이 코드는 최종 연산이 없기 때문에 실제로 실행되지 않는다.

    ```java
    names.stream()
            .filter(name -> name.startsWith("k"))
            .toList();
    ```

  이렇게 `toList()` 같은 최종 연산이 있어야 실행된다.
    
  ---

  ## 실무에서 많이 쓰는 예시

  엔티티 리스트를 DTO 리스트로 바꿀 때 많이 쓴다.

    ```java
    List<MemberResponse> responses = members.stream()
            .map(member -> MemberResponse.from(member))
            .toList();
    ```

  조건에 맞는 데이터만 뽑을 때 사용한다.

    ```java
    List<Mission> successMissions = missions.stream()
            .filter(mission -> mission.isSuccess())
            .toList();
    ```

  특정 값만 추출할 때 사용한다.

    ```java
    List<String> names = members.stream()
            .map(Member::getName)
            .toList();
    ```
    
  ---

  ## 반복문과 비교

  반복문으로 쓰면 이렇게 된다.

    ```java
    List<String> names = new ArrayList<>();
    
    for (Member member : members) {
        if (member.getAge() >= 20) {
            names.add(member.getName());
        }
    }
    ```

  Stream으로 쓰면 이렇게 된다.

    ```java
    List<String> names = members.stream()
            .filter(member -> member.getAge() >= 20)
            .map(Member::getName)
            .toList();
    ```

- 객체 그래프 탐색

  ### 1. 객체 그래프(Object Graph)란?

  프로그램에서 객체들이 서로를 참조하며 형성하는 구조 전체를 말함.

  예시:

    ```java
    class Member {
        Profile profile;
        List<Order> orders;
    }
    
    class Order {
        Delivery delivery;
    }
    ```

  `Member → orders → Order → delivery`

  이렇게 연결된 모든 객체들의 네트워크가 객체 그래프.
    
  ---

  ### 2. 객체 그래프 탐색(Object Graph Traversal)

  객체 그래프 탐색은 최초 객체에서 시작해 참조를 타고 들어가며 필요한 데이터를 ‘깊이’ 탐색하는 과정이다.

  예:

    ```java
    Member member = findMember();
    Delivery d = member.getOrders().get(0).getDelivery();
    ```

  → Member에서 Orders로

  → Orders에서 Order로

  → Order에서 Delivery로

  이렇게 객체를 “따라 들어가는” 것이 그래프 탐색.
    
  ---

  ### 3 . JPA에서의 객체 그래프 탐색 핵심 개념

    - 프록시 초기화
        - `member.getOrders()` 호출 시 Hibernate는 **프록시를 통해 필요할 때 로딩**함.
    - FetchType.LAZY / EAGER
        - **LAZY**: 접근하는 순간 쿼리 실행
        - **EAGER**: Member 로딩 시 Order도 같이 로딩

      → 결국 탐색 과정이 *쿼리 실행 흐름*에 직접 영향.

    - N+1 문제

      객체 그래프를 잘못 순회하면 반복적으로 DB 조회 발생.


### 4. 한 줄 요약
    JPA에서 객체 그래프 탐색이란, 엔티티 간 연관관계를 통해 객체 필드를 따라가면서 데이터를 접근하는 것이며, 이 과정에서 로딩 전략(LAZY/EAGER)에 따라 SQL 쿼리가 실행되고 이는 성능, 순환 참조(무한 루프), N+1 문제 등 다양한 결과를 낳는다.
    

- @Valid vs @Validated

  ## 1. 개념

  `@Valid`는 객체에 선언된 Bean Validation 규칙을 실행하라는 트리거 역할을 하는 어노테이션이다.

  DTO의 각 필드에 붙어 있는 `@NotBlank`, `@Size`, `@Email` 등의 검증 규칙을 기반으로 값의 유효성을 검사한다.

  ## 2. @Valid의 사용 위치

  ### 2-1. 컨트롤러 파라미터 DTO

  요청 본문(`@RequestBody`)이나 요청 파라미터(`@ModelAttribute`)로 DTO를 받을 때 `@Valid`를 함께 사용한다.

    ```java
    @PostMapping("/members")
    public ResponseEntity<?> createMember(@Valid @RequestBody MemberRequest request) {
        ...
    }
    ```

    - JSON 바디 바인딩 후 검증 실패 시 `MethodArgumentNotValidException`이 발생한다.
    - `@ModelAttribute` 기반 요청이면 `BindException`이 발생한다.

    ---

  ### 2-2. 중첩 객체(Nested DTO) 검증

  DTO 내부에 또 다른 DTO가 있을 경우, 내부 DTO 필드에 `@Valid`를 붙여야 내부 필드의 검증도 함께 수행된다.

    ```java
    public class OrderRequest {
    
        @NotNull
        private Long memberId;
    
        @Valid
        private Address address;
    }
    ```
    
  ---

  ## 3. 검증 실패 시 처리 흐름

  ### 3-1. BindingResult 없이 @Valid 사용

  컨트롤러 파라미터에 `BindingResult`를 받지 않을 경우, 검증 실패 시 스프링이 곧바로 예외를 던진다.

    - `@RequestBody` → `MethodArgumentNotValidException`
    - `@ModelAttribute` → `BindException`

  이 예외는 보통 `@RestControllerAdvice`에서 공통으로 처리한다.
    
  ---

  ### 3-2. BindingResult와 함께 사용

  검증 대상 파라미터 **바로 뒤에** `BindingResult`를 선언하면 예외가 발생하지 않고 컨트롤러 내부에서 검증 결과를 직접 확인할 수 있다.

    ```java
    @PostMapping("/members")
    public String create(@Valid MemberForm form, BindingResult bindingResult) {
        if (bindingResult.hasErrors()) {
            return "members/new-form";
        }
        ...
    }
    ```

  `BindingResult`는 필드 에러, 객체 에러를 포함하며 `hasErrors()`로 간단하게 검증 실패 여부를 알 수 있다.
    
  ---

  ## 4. 에러 메시지 처리

  검증 어노테이션에 message 값을 직접 작성하거나, 메시지 파일(`messages.properties`)을 통해 관리할 수 있다.

    ```java
    @NotBlank(message = "{member.name.notBlank}")
    private String name;
    ```

    ```
    member.name.notBlank=이름은 필수 입력값이다.
    ```

  Spring의 MessageSource가 코드 → 메시지로 변환해준다.
    
  ---

  ## 5. @Valid vs @Validated

  ### @Valid

    - Bean Validation 표준 어노테이션이다.
    - 객체 검증만 수행하며 **검증 그룹(group)** 기능은 없다.

  ### @Validated

    - 스프링이 제공하는 확장 어노테이션이다.
    - 그룹 검증을 지원한다.
    - 특정 상황(Create, Update 등)에 따라 다른 검증 규칙을 적용할 때 사용한다.

    ```java
    @PostMapping("/members")
    public ResponseEntity<?> create(@Validated(Create.class) @RequestBody MemberRequest req) {
        ...
    }
    
    public interface Create {}
    public interface Update {}
    
    public class MemberRequest {
    
        @NotBlank(groups = Create.class)
        private String name;
    
        @NotNull(groups = Update.class)
        private Long memberId;
    }
    
    // 생성 API에서는 name만 검사하고 싶고, 수정 API에서는 memberId만 검사하고 싶을 때
    
    @PostMapping("/members")
    public void createMember(@Validated(Create.class) @RequestBody MemberRequest request) {
    }
    
    @PatchMapping("/members")
    public void updateMember(@Validated(Update.class) @RequestBody MemberRequest request) {
    }
    ```