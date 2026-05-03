미션 수행한 깃허브 브랜치: [feature/#6](https://github.com/choehyeonjin/UMC-10th-spring-study/tree/feature/%236)

1. 엔티티를 제작하고 매핑까지 완료하기

   ![erd](week06/mission/mission_06_(1).png)

    - 모든 엔티티가 BaseEntity를 상속하게 하여 데이터 추적을 용이하게 했다.
    - @Column으로 길이 제한, nullable 등 제약 사항을 설정했다.
    - @Builder.Default로 기본값을 설정했다.
    - 약관 본문, 리뷰 본문과 같은 필드에는 @Lob을 적용했다.
        - 별도의 설정이 없으면 JPA는 Java의 String을 DB의 VARCHAR(255)로 매핑한다.
          @Lob을 붙이면 JPA는 이를 TEXT로 매핑한다.
    - 생명주기를 함께 관리하기 위해 양방향 매핑은 다음과 같이 설정했다.
        - Member - MemberFood (1:N)
        - Member - MemberTerm (1:N)
        - Review - ReviewReply (1:1)
        - Review - ReviewImage (1:N)
        - Inquiry - InquiryReply (1:1)
        - Inquiry - InquiryImage (1:1)
        - Store - Mission (1:N) 등 자식 데이터가 많아 객체 탐색에 메모리 부하가 유발될 수 있다고 생각해, 대용량 데이터나 페이징이 필요한 조회에는 레포지토리 기반 조회를 채택했다.
2. 아래 화면을 구현하기 위한 서비스를 만들고 5주차에 제작한 컨트롤러와 연결하기
    - 각 도메인별 커스텀 에러코드를 추가하여 예외처리했다.
    - 다음 API들을 구현했다.
        - 리뷰 작성
            - API 명세서
                - API 개요
                    - 특정 가게에 리뷰를 작성하는 API입니다.
                    - 별점(0.5~5.0), 리뷰 내용, 첨부 이미지 ID(최대 3개)를 저장할 수 있습니다.
                    - 리뷰 이미지 첨부시 별도 이미지 업로드 API를 사용합니다.
                - API Endpoint
                    - `POST /reviews`
                - Request Header

                ```json
                Authorization: Bearer {accessToken}
                Content-Type: application/json
                ```

                - Request Body

                ```json
                {
                  "storeId":31,
                  "rating":4.5,
                  "content":"맛있었습니다.",
                  "imageIds": [12,13]
                }
                ```

                - Success Response

                ```json
                {
                  "isSuccess":true,
                  "code":"REVIEW201_1",
                  "message":"리뷰 작성에 성공했습니다.",
                  "result": {
                    "reviewId":209,
                    "createdAt":"2026-03-21T12:34:56"
                  }
                }
                ```
            
            ![image.png](week06/mission/mission_06_(2).png)
            
        - 미션 목록 조회 (진행중, 진행완료)
            - API 명세서
                - API 개요
                    - 미션 목록을 조회하는 API입니다.
                    - Query String으로 진행중, 진행완료인 미션을 필터링할 수 있습니다.
                    - 미션 포인트, 가게 이름, 미션 조건 조회
                - API Endpoint
                    - `GET /missions`
                - Request Header
                    
                    ```json
                    Authorization: Bearer {accessToken}
                    ```
                    
                - Query String
                    
                    | Key | Type | Required | Description |
                    | --- | --- | --- | --- |
                    | status | String | O | 조회할 미션 상태 (`IN_PROGRESS`, `SUCCESS`) |
                    | page | Integer | X | 페이지 번호, 기본값 0 |
                    | size | Integer | X | 페이지당 개수, 기본값 3 |
                - Success Response
                    - 미션 성공 후 리뷰 작성 가능 여부 및 리뷰 작성 상태를 함께 조회하기 위해 리뷰 작성 여부reviewWritten를 응답에 포함합니다.
                    
                    ```json
                    {
                      "IsSuccess":true,
                      "code":"MISSION200_1",
                      "message":"미션 목록 조회에 성공했습니다.",
                      "result": {
                        "pageInfo": {
                          "page":0,
                          "size":3,
                          "totalElements":142,
                          "totalPages":48,
                          "hasNext":true
                        },
                        "missions": [
                          {
                            "memberMissionId":831,
                            "missionId":8,
                            "storeId":31,
                            "status":"IN_PROGRESS",
                            "storeName":"가게이름a",
                            "missionPoint":500,
                            "missionCondition":"12,000원 이상의 식사",
                            "reviewWritten":false
                          }
                        ]
                      }
                    }
                    ```
                - 커스텀 예외 처리 확인
                ![image.png](week06/mission/mission_06_(3).png)
                - 진행중
                ![image.png](week06/mission/mission_06_(4).png)
                - 진행완료
                ![image.png](week06/mission/mission_06_(5).png)
            
        - 마이페이지 조회
            - API 명세서
                - API 개요
                    - 마이페이지 화면 조회 API입니다.
                - API Endpoint
                    - `GET /members/me`
                - Request Header
                    
                    ```json
                    Authorization: Bearer {accessToken}
                    ```
                    
                - Success Response
                    
                    ```json
                    {
                      "IsSuccess": true,
                      "code": "MYPAGE200_1",
                      "message": "마이페이지 조회에 성공했습니다.",
                      "result": {
                        "profileImageUrl": null, 
                        "nickname": "nickname012",
                        "email": "email012@naver.com",
                        "phoneNumber": null,
                        "isPhoneVerified": false,
                        "point": 2500
                      }
                    }
                    ```
                  ![image.png](week06/mission/mission_06_(6).png)
            
        - 홈 화면 조회
            - API 명세서
                - API 개요
                    - 홈 화면 조회 API입니다. 
                    Query String인 regionId로 지역을 변경하고 필터링할 수 있습니다.
                    1. 지역 선택 정보
                        - 지역이름, 해당 지역에서 완료한 미션 개수 조회
                    2. 선택된 지역에서 도전 가능한 미션 목록 (MY MISSION)
                        - 미션 개요(미션 포인트, 미션 조건) 미션 기한, 가게 이름, 가게 카테고리 조회
                - API Endpoint
                    - `GET /home`
                - Request Header
                    
                    ```json
                    Authorization: Bearer {accessToken}
                    ```
                    
                - Query String

                    | Key | Type | Required | Description |
                    | --- | --- | --- | --- |
                    | regionId | Long | O | 선택된 지역 ID |
                    | page | Integer | X | 페이지 번호, 기본값 0 |
                    | size | Integer | X | 페이지당 개수, 기본값 3 |
                - Success Response
                    
                    ```json
                    {
                      "IsSuccess":true,
                      "code":"HOME200_1",
                      "message":"홈 화면 조회에 성공했습니다.",
                      "result": {
                        "region": {
                          "regionId":3,
                          "regionName":"안암동",
                          "completedMissionCount":7
                        },
                        "pageInfo": {
                          "page":0,
                          "size":3,
                          "totalElements":12,
                          "totalPages":4,
                          "hasNext":true
                        },
                        "missions": [
                          {
                            "missionId":8,
                            "storeId":31,
                            "storeName":"반이학생마라탕",
                            "storeCategory":"중식당",
                            "missionPoint":500,
                            "missionCondition":"10,000원 이상의 식사",
                            "missionDeadline":"2026-03-29",
                            "deadlineLabel":"D-7"
                          }
                        ]
                      }
                    }
                    ```
              ![image.png](week06/mission/mission_06_(7).png)