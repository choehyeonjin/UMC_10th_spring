## 아키텍처 구조란?
- **소프트웨어를 어떤 기준으로 나누고, 각 부분을 어떻게 연결할지에 대한 설계**
- 공통적인 아키텍처 구조 기본 원칙
    - 단일 책임 원칙: 각 객체, 계층은 하나의 역할만 부여
      (요청을 받고 응답을 전달, 비즈니스 로직을 처리, DB에 저장하는 로직)
    - 모듈화:
      하나의 기능을 담은 코드를 모듈로 관리
      다른 객체, 계층에서 해당 기능이 필요한 경우 해당 모듈에게 지시
    - 낮은 의존성:
      한 부분의 변화가 다른 부분에 큰 영향을 미치지 않게 설계
- 정의 요소
    1. 분리 → 코드를 어떤 기준으로 나눌 것인가?
        - 역할 기준 → Controller / Service / Repository
        - 기능 기준 → User / Order / Auth
        - **→ 계층/도메인형 구조**
    2. 흐름 → 각 부분이 어떻게 연결되고 호출되는가?
        - Controller → Service → Repository
        - 외부 요청 → 내부 로직 → DB
- **아키텍처 구조를 설계할 때는 지금 프로젝트와 해당 구조가 적절한지, 장단점이 무엇인지, 지금 상황을 봤을때 이렇게 설계하는게 맞는지 고려해야 한다.**

## Swagger란?
- 백엔드 API를 문서화하고, 직접 테스트까지 할 수 있게 해주는 도구
- Swagger를 사용하는 이유

    1. API 문서 자동 생성

    - 코드에 어노테이션만 달면 자동으로 API 명세서를 만들어준다.
    - 파일이나 노션에 일일이 파라미터를 적을 필요가 없어, 코드와 문서가 불일치하는 문제를 차단
    - Endpoint, Method, Request/Response 구조 등을 모두 포함

    2. 직접 테스트 가능

    - 브라우저에서 바로 요청 보내고 응답 확인 가능
    - Postman 없이도 테스트 가능
    3. 협업 효율 극대화
    - 백엔드 개발자가 Swagger 주소만 넘겨주면,
      프론트엔드 개발자는 해당 문서를 보고 바로 연동 작업을 시작
## 도메인형 아키텍처란?

### 구조

```
    com.example.umc10th.domain
     ┣ member
     ┃ ┣ controller
     ┃ ┣ service
     ┃ ┣ entity
     ┃ ┗ repository
     ┣ mission
     ┃ ┣ controller
     ┃ ┣ service
     ┃ ┗ entity
     ┗ review
   ```

### 특징

- 도메인(ERD 기준 Entity 그룹) 단위로 패키지를 구성
- 각 도메인 패키지 내부에서 Controller / Service / Entity / Repository를 모두 포함
- DDD(Domain-Driven Design)의 기초적인 형태

### 장점

  | 항목 | 설명 |
    | --- | --- |
  | ERD 기반 도메인 매핑 용이 | Member, Mission 등 엔티티 단위로 구조가 그대로 반영됨 |
  | 협업 시 역할 분담 명확 | 각 팀원이 도메인 단위로 맡아 개발 가능 → 충돌 최소화 |
  | 확장성 높음 | 신규 도메인 추가 시, 기존 코드에 영향 없이 패키지 추가만으로 확장 가능 |
  | 코드 응집도 향상 | 관련된 비즈니스 로직과 엔티티가 같은 패키지에 위치하여 관리 용이 |
  | 리뷰와 디버깅 용이 | “Member” 관련 기능은 member 폴더만 보면 됨 |

### 단점

  | 항목 | 설명 |
    | --- | --- |
  | 초기 구조 복잡 | 패키지가 많아지고, 초반엔 구조가 낯설 수 있음 |
  | 중복 가능성 | 공통 로직/유틸을 각 도메인에서 중복 구현할 수 있음 (→ global 패키지로 분리 필요) |
  | DDD 개념 이해 필요 | 도메인 간 경계나 의존성 설정에 대한 설계 고민이 필요함 |

## DDD vs 도메인형 아키텍처

### 1. 도메인형 아키텍처 (Package by Domain)

- **코드를 기능(도메인) 기준으로 묶는 구조**
- DDD의 핵심 아이디어를 빌려와서 **폴더 구조**를 짜는 방식

### 2. DDD (Domain-Driven Design)

[참고자료](https://strong-park.tistory.com/entry/DDD-%EC%9E%85%EB%AC%B8%EC%84%9C%EB%A5%BC-%EC%9D%BD%EA%B3%A0-%EB%82%98%EC%84%9C-%EB%8A%90%EB%82%80-DDD%EB%9E%80-%EB%AC%B4%EC%97%87%EC%9D%B8%EA%B0%80)

  - 훨씬 큰 개념 → **비즈니스(도메인)를 중심으로 소프트웨어를 설계하는 방법론**
  - 기술적인 코드보다 **비즈니스 용어(Ubiquitous Language)와 업무 규칙**을 가장 중요하게 생각
  - 전체 서비스를 여러 개의 **Bounded Context**(독립적인 경계)로 나누고, 각 경계 간의 관계를 정의 (예: 주문 문맥 vs 배송 문맥)
  - 코드 내부에서 데이터를 담는 **Entity**, 식별자가 없는 **Value Object(VO)**, 로직을 응집하는 **Aggregate**, 외부 통로인 **Repository** 등으로 역할을 엄격히 나눔
  - Aggregate vs Bounded Context

    ![image1](/week04/keyword/keyword_04_(1).png)

      - 예시) “상품”
          - Aggregate의 “상품”: 해당 도메인에서 “상품”에 대한 정보나 규칙을 관리하는 객체들이 하나의 트랜잭션으로 묶여 관리된다.
            예) 상품 명, 상품 가격, 상세 설명을 업데이트하는 로직
          - Bounded Context의 “상품”: 하위 도메인마다 다른 의미를 가질 수 있다.
            예) 카탈로그 도메인에서는 “상품”이 상품 이미지, 설명, 가격 등 마케팅과 관련된 정보로 정의
            재고 관리 도메인에서는 “상품”이 재고를 추적하기 위한 물리적으로 존재하는 상품을 관리하는 용어

## 왜 DTO를 사용하는가?
- DTO (Data Transfer Object)는 **데이터를 계층 간에 전달하기 위한 객체이다.**
- 엔티티를 직접 노출하지 않기 위해, 필요한 데이터만 주고받기 위해 사용한다.
  1. 엔티티 보호

      ```java
      public class UserResponseDto {
          private String name;
          @NotBlank
          private String email;
      }
      ```

      - 엔티티를 그대로 반환 시 비밀번호와 같은 민감한 정보까지 노출될 수 있어 위험하다.
      - validation 걸기에 용이하다.
  2. API 스펙과 DB 구조 분리

      ```java
      // 엔티티
      class User {
          String username;
          String password;
          LocalDateTime createdAt;
      }
        
      // DTO
      class UserResponseDto {
          String name;   // username → name으로 변경 가능
          String previewText; // 가공된 데이터
      }
      ```

      - DB랑 API를 독립적으로 설계 가능
      - 가공된 데이터 전달 가능
  3. 계층 간 역할 분리

      ```
      Controller → DTO
      Service → Entity
      Repository → Entity
      ```

  4. 요청/응답 데이터 분리
    
    ```java
    class UserRequestDto {
        String email;
        String password;
    }
    
    class UserResponseDto {
        String id;
        String name;
    }
    ```
    
    - 입력값 / 출력값을 다르게 설계 가능

## 컨버터는 왜 사용하는가?
- 컨버터는 객체를 다른 객체로(보통 Entity ↔ DTO) 변환해주는 역할을 한다.
- 계층 간 데이터 형태를 맞춰주기 위한 변환 도구
- 변환 로직을 한 곳으로 모을 수 있음

    ```java
    // 컨버터 X
    public UserResponseDto getUser() {
        User user = userRepository.findById(1L);
        
        return new UserResponseDto(
            user.getName(),
            user.getEmail()
        );
    }
        
    // 컨버터 O
    public UserResponseDto getUser() {
        User user = userRepository.findById(1L);
        return UserConverter.toDto(user);
    }
    ```

    - 코드 중복 제거
    - 계층 간 역할 분리
    - 엔티티 보호