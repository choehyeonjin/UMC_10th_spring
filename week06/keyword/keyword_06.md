## JPA란?

   1. JPA란?

  JPA(Java Persistence API)는 자바에서 객체(Object)와 관계형 데이터베이스(Relational DB) 사이의 데이터를 매핑(ORM, Object-Relational Mapping) 해주는 기술 표준이다.

  즉, SQL을 직접 작성하지 않고 자바 객체 중심으로 데이터베이스를 다룰 수 있게 해주는 ORM 표준 인터페이스이다.
    
  ---

   2. ORM이란?

  ORM(Object Relational Mapping)은 객체(Object) 와 데이터베이스의 테이블(Table) 을 매핑하여

  SQL 대신 객체 지향적인 방식으로 데이터 조작을 가능하게 하는 기술이다.
    
  ---

   3. JPA 사용 시 개발 흐름

    1. Entity 정의 (`@Entity`, `@Table`, `@Id` 등)
    2. Repository 생성 (Spring Data JPA 활용)
    3. Service에서 트랜잭션 관리 및 비즈니스 로직 작성
    4. Controller에서 Service 호출
    5. JPA 구현체(대표적으로 Hibernate)가 SQL 자동 생성 및 DB 반영
## N+1 문제란?

   1. 개념

  JPA에서 연관된 엔티티를 지연 로딩(LAZY) 으로 조회할 때, 한 번의 조회(1) 이후 연관된 엔티티를 각각 N번 추가 조회하는 비효율적 쿼리 현상.

  > 부모 1번 + 자식 N번 → 총 N+1개의 SQL 실행
  >
    
  ---

2. 예시

   ```java
    List<Member> members = memberRepository.findAll();
    for (Member m : members) {
        System.out.println(m.getMissions().size());
    }
   ```

→ 실행 SQL

   ```sql
    SELECT * FROM member;              -- 1번
    SELECT * FROM mission WHERE member_id = 1;  -- N번
   ```
    
  ---

3. 원인

    - `@OneToMany`, `@ManyToOne` 관계의 **LAZY 로딩**
    - 부모만 먼저 조회하고, 자식은 접근할 때마다 쿼리 발생

---


4. 해결 방법

  | 방법 | 설명 |
      | --- | --- |
  | **Fetch Join** | 한 번의 JOIN으로 연관 데이터 함께 조회 (`JOIN FETCH`) |
  | **EntityGraph** | `@EntityGraph(attributePaths = {"missions"})` 사용, JPQL 없이 fetch join |
  | **Batch Size 설정** | `hibernate.default_batch_fetch_size: 100` → 여러 건 묶어서 조회 |
  | **EAGER 로딩** | 항상 함께 로딩하지만, 불필요한 데이터까지 가져와 비추천 |
## 지연로딩과 즉시로딩의 차이는?
- 로딩: 엔티티를 언제 로딩할 지에 대한 것
- 즉시 로딩(EAGER): 엔티티를 로드할 때 연관 엔티티까지 즉시 함께 조회.

    ```java
    @Entity
    @Table(name = "member_food")
    public class MemberFood {
        @ManyToOne(fetch = FetchType.EAGER) @JoinColumn(name = "member_id")
        private Member member;
        @ManyToOne(fetch = FetchType.EAGER) @JoinColumn(name = "food_id")
        private Food food;
    }
    ```

    - findAll() 시점에 member, food까지 즉시 조인/조회.
    - 즉시 데이터 접근이 가능.
    - 목록에서 거의 쓰지 않는 연관까지 매번 당겨 과다 데이터·조인 비용 증가.
    - 쿼리 최적화 관련하여 눈에 보이지 않는 쿼리까지 발생.
  - 지연 로딩(LAZY): 연관 엔티티는 프록시로 두고, 실제로 접근할 때 쿼리 실행.

      ```java
      @Entity
      @Table(name = "member_food")
      public class MemberFood {
          @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "member_id")
          private Member member;
          @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "food_id")
          private Food food;
      }
      ```

      - 초기에는 MemberFood만 SELECT. member.getName() 등 접근 시점에 쿼리 발행.
      - 처음에는 불필요한 데이터 로딩 방지
## JPQL란?

**JPQL (Java Persistence Query Language)**

- 개념
        - JPA의 객체 지향 쿼리 언어
        - SQL처럼 보이지만, 테이블이 아닌 엔티티(Entity) 와 필드를 대상으로 동작함
        - 즉, 데이터베이스 중심이 아니라 객체 중심의 쿼리를 작성하게 도와줌
        - SQL은 데이터베이스 테이블을 대상으로,
          JPQL은 영속성 컨텍스트에서 관리되는 엔티티 객체를 대상으로 함
- 예시

    ```java
    @Query("select m from Member m where m.name = :name")
    List<Member> findByName(@Param("name") String name);
    ```

- 사용법
    1. 메서드 이름 기반 쿼리 생성
        - Spring Data JPA가 메서드 이름을 해석해 JPQL 자동 생성

            ```java
            List<Member> findByNameAndDeletedAtIsNull(String name);
            ```

        - 자동 실행되는 JPQL

            ```java
            select m from Member m where m.name = :name and m.deletedAt is null
            ```

            - 장점: 간단한 조건이면 SQL을 직접 쓸 필요 없음
            - 단점: 복잡한 조건일수록 메서드 이름이 너무 길어짐
    2. @Query 어노테이션으로 직접 작성
        - 복잡한 조건이나 커스터마이징이 필요할 때 사용

            ```java
            @Query("select m from Member m where m.name = :name and m.deletedAt is null")
            List<Member> findActiveMember(@Param("name") String name);
            ```

            - :파라미터명으로 변수 바인딩
            - nativeQuery = true 옵션으로 SQL 직접 실행도 가능

- 주의할 점

  - JPQL 자동 생성 쿼리가 항상 최적화되어 있진 않음 → 실행 쿼리 로그 꼭 확인
  - 연관관계 로딩 시 N+1 문제가 생길 수 있음
  - 복잡한 조건, 동적 쿼리, 타입 안정성이 필요한 경우엔 QueryDSL 사용이 더 적합
## Fetch Join란?
- 개념
      - JPQL에서 연관된 엔티티를 한 번의 쿼리로 함께 조회하기 위한 방법
        → 지연 로딩 쿼리에서 N+1 문제를 해결할 때 가장 많이 쓰임
      - JPQL에서 `join fetch` 문법을 사용하면, 연관된 엔티티를 즉시 로딩(EAGER)하되, Hibernate가 강제로 JOIN으로 한 번에 가져옴

    ```java
      @Query("select m from Member m join fetch m.memberFoodList")
      List<Member> findAllWithFoods();
    ```

- 특징

    | 항목 | 설명 |
        | --- | --- |
        | **의미** | 연관 엔티티를 즉시 함께 가져옴 (조인해서 한 번에) |
        | **장점** | 쿼리 1번으로 필요한 데이터 전부 조회 가능 |
        | **활용 예시** | 상세 조회 등 연관 엔티티가 꼭 필요한 경우 |
## @EntityGraph란?
- 개념
- JPA에서 fetch join을 어노테이션으로 선언형으로 표현할 수 있는 기능
  → JPQL 없이도 특정 연관 엔티티를 함께 로딩하도록 설정 가능.
- Fetch Join과 비슷한 기능을 가지지만, JPQL 쿼리를 직접 쓰지는 않고, 레포지토리 메서드 레벨에서 조정할 수 있는 선택적 즉시 로딩 전략

    ```java
        @EntityGraph(attributePaths = {"memberFoodList"})
        @Query("select m from Member m")
        List<Member> findAllWithFoods();
    ```

    - 특징

        | 항목 | 설명 |
        | --- | --- |
        | **목적** | fetch join과 동일하게 N+1 방지 |
        | **차이점** | 코드보다 선언적으로 작성, 유지보수 쉬움 |
        | **적합한 경우** | 단순 조회나 기본 Repository 메서드 재사용 시 |
        | **주의** | join 조건 제어 불가능 (fetch join보다 제약 있음) |
        
        | 기능 | Fetch Join | EntityGraph |
        | --- | --- | --- |
        | 작성 방식 | JPQL 직접 작성 | 어노테이션 기반, 선언적 |
        | 적용 위치 | 쿼리 단위 | Repository 메서드 단위 |
        | 유지보수 | JPQL 수정 필요 | 엔티티 필드명만 유지하면 됨 |
## commit과 flush 차이점은?
- 개념
   - flush - 영속성 컨텍스트의 변경 내용을 DB에 SQL로 반영만 하는 작업
    - commit - 트랜잭션을 종료하고, DB에 반영된 내용을 확정하는 작업

    트랜잭션 커밋 과정에서 JPA는 내부적으로 **flush → commit** 순으로 실행됨.

  | 구분 | flush | commit |
                  | --- | --- | --- |
          | **의미** | 영속성 컨텍스트의 변경 내용을 DB에 반영 (SQL 실행) | 트랜잭션을 실제로 종료하고 확정 |
          | **시점** | 커밋 직전 자동 실행 또는 직접 호출 `em.flush()` 가능 | flush 이후 실행됨 |
          | **예시** | 변경 감지(Dirty Checking) 후 update 쿼리 전송 | DB에 변경사항 영구 반영 |
   
- 어차피 flush가 반영되는데, 수동으로 왜 사용할까?

    1. 중간에 SQL을 강제로 DB에 반영하고 싶은 경우
    2. 벌크 연산 후 영속성 컨텍스트와 DB 상태를 맞추고 싶은 경우
    3. JPQL 실행 전에 명확한 상태로 만들어야 할 때