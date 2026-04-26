- 미션 수행한 깃허브 브랜치:
  [feature/#5](https://github.com/choehyeonjin/UMC-10th-spring-study/tree/feature/%235)

- API 명세서
    - 2주차 미션으로 작성한 API 명세서를 피드백을 반영하여 아래와 같이 수정하였다.
      - 홈 화면 조회 API에 페이징 관련 쿼리 스트링을 추가했다.
      - 홈 도메인은 미션, 지역 등 특정 도메인에 종속되지 않는다고 생각하여 home 패키지를 생성하였다.
      - 미션 도전 API를 추가했다.
      - 미션 성공 처리 API 메서드를 PATCH로 변경하였다.
      - 도메인 별 성공 코드를 반영하여 성공 응답 구조를 작성하였다.
    
    - 홈 화면 조회
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
            
    - 리뷰 작성
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
        
    - 미션 목록 조회(진행중, 진행 완료)
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
            - 미션 성공 후 리뷰 작성 가능 여부 및 리뷰 작성 상태를 함께 조회하기 위해 리뷰 작성 여부 reviewWritten를 응답에 포함합니다.
            
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
            
    - 미션 도전
        - API 개요
            - 사용자가 특정 미션에 도전하는 API입니다.
            - 호출 성공 시 `memberMissionId`가 생성됩니다.
            - 이후 미션 성공 처리는 `memberMissionId`를 기준으로 진행합니다.
        - API Endpoint
            - `POST /missions/{missionId}/challenge`
        - Request Header
            
            ```json
            Authorization: Bearer {accessToken}
            ```
            
        - Path Variable
            
            | Key | Type | Description |
            | --- | --- | --- |
            | missionId | Long | 도전할 미션 ID |
        - Success Response
            
            ```json
            {
              "IsSuccess":true,
              "code":"MISSION201_1",
              "message":"미션 도전에 성공했습니다.",
              "result": {
                "memberMissionId":831,
                "missionId":8,
                "status":"IN_PROGRESS",
                "createdAt":"2026-03-21T12:34:56"
              }
            }
            ```
            
    - 미션 성공 처리
        - API 개요
            - 진행 중인 미션을 성공 상태로 변경하는 API입니다.
        - API Endpoint
            - `PATCH /member-missions/{memberMissionId}/success`
        - Request Header
            
            ```json
            Authorization: Bearer {accessToken}
            Content-Type: application/json
            ```
            
        - Path Variable
            
            | Key | Type | Description |
            | --- | --- | --- |
            | memberMissionId | Long | 성공 처리할 사용자 미션 ID |
        - Success Response
            
            ```json
            {
              "IsSuccess":true,
              "code":"MISSION200_2",
              "message":"미션 성공 처리에 성공했습니다.",
              "result": {
                "memberMissionId":831,
                "status":"SUCCESS",
                "updatedAt":"2026-03-21T12:40:10"
              }
            }
            ```
            
    - 회원가입
        - API Endpoint
            - `POST /auth/signup`
        - Request Header
            
            ```json
            Content-Type: application/json
            ```
            
        - Request Body
            
            ```json
            {
              "termIds": [1,2,3],
              "name":"최현진",
              "gender":"FEMALE",
              "birthdate":"2004-08-31",
              "address":"서울시 노원구 월계동",
              "preferredFoodIds": [1,2,4]
            }
            ```
            
        - Success Response
            
            ```json
            {
              "IsSuccess":true,
              "code":"MEMBER201_1",
              "message":"회원가입에 성공했습니다.",
              "result": {
            	  "memberId": 1
                "createdAt":"2026-01-09T12:00:00"
              }
            }
            ```