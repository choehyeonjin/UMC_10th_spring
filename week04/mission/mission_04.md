### ERD 사진

![image1](week04/mission/mission_04_(1).png)

### 도메인형 아키텍처 세팅
- 미션 수행한 깃허브 브랜치
    - [feat/chapter4](https://github.com/choehyeonjin/UMC-10th-spring-study/tree/feat/chapter4)

- 설계한 ERD를 바탕으로 큰 도메인을 다음과 같이 다섯 가지로 구분하였다.
    - 사용자(Member, MemberFood, Food, MemberTerm, Term)
    - 미션(Mission, MemberMission, Store, Region)
    - 리뷰(Review, ReviewPhoto, ReviewReply)
    - 문의(Inquiry, InquiryPhoto, InquiryReply)
    - 이미지(Image)

- 도메인형 아키텍처로 세팅한 패키지 구조
    - 하나의 도메인에 controller, converter, entity, enums, exception, repository, service로 구성했다.
    - 해당 도메인이 제공하는 기능을 한눈에 파악하고자 Service 인터페이스와 ServiceImpl 구현체로 나누었다.
    - 추후 UI 요구사항과 비즈니스 로직 요구사항을 분리하기 위해, DTO를 request/response/result 패키지로 구분하는 것도 고려 중이다.
  ![image2](week04/mission/mission_04_(2).png)
  ![image3](week04/mission/mission_04_(3).png)
  ![image4](week04/mission/mission_04_(4).png)
  ![image5](week04/mission/mission_04_(5).png)
  ![image6](week04/mission/mission_04_(6).png)