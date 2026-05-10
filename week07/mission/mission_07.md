- 미션 수행한 깃허브 브랜치: https://github.com/choehyeonjin/UMC-10th-spring-study/tree/feature/%237
1. 제네릭을 활용한 공통 페이징 응답 객체 도입(오프셋 기반)
    - 기존 페이징 API들은 페이징 메타데이터(`PageInfoDTO`)만 공통으로 사용하고, 데이터를 감싸는 응답 객체는 도메인별로 개별 생성해야 했다.
    - 이번 주차 워크북 내용을 바탕으로, 제네릭을 적용한 페이지네이션 틀 `PageResDTO<T>`를 도입하여 실제 조회 데이터는 `data` 필드에, 페이징 메타데이터는 `pageInfo` 필드에 담도록 구조화했다.
    - 페이징 조회 API(홈 화면 조회, 미션 목록 조회)가 이 공통 DTO를 사용하게 되어 응답 포맷의 일관성을 확보하고 코드 재사용성을 향상시켰다.

    ```java
    package com.example.umc10th.global.dto;
    
    import lombok.Builder;
    import java.util.List;
    
    @Builder
    public record PageResDTO<T>(
            List<T> data,
            PageInfoDTO pageInfo
    ) {
        @Builder
        public record PageInfoDTO(
                Integer page,
                Integer size,
                Long totalElements,
                Integer totalPages,
                Boolean hasNext
        ) {}
    }
    ```
---
2. 내가 진행중인 미션 조회하기
    - Query String으로 진행중/진행완료인 미션을 필터링하고 오프셋 기반 페이지네이션으로 응답하는 미션 목록 조회 API `GET /missions` 가 기존에 구현되어 있어,
      기존 memberId가 하드코딩 되어 있던 것에서 Request Body에서 받는 것으로 수정하였다.
    - HTTP 표준에 따르면, `GET` 요청은 원칙적으로 Request Body를 포함하지 않는 것을 권장한다고 하여 요청 메서드를 `POST`로 변경하였다.
    - 기존 컨트롤러

        ```java
        // 미션 목록 조회
            @GetMapping("/missions")
            public ApiResponse<PageResDTO<MissionResDTO.MissionDetailDTO>> getMissions(
                    @RequestParam String status,
                    @RequestParam(defaultValue = "0") Integer page,
                    @RequestParam(defaultValue = "3") Integer size
            ) {
                Long memberId = 1L; // TODO: 인증 연동
        
                PageResDTO<MissionResDTO.MissionDetailDTO> resultDTO = missionService.getMyMissions(memberId, status, page, size);
        
                return ApiResponse.onSuccess(
                        MissionSuccessCode.OK,
                        resultDTO
                );
            }
        ```

    - 변경 컨트롤러

        ```java
        // 미션 목록 조회
            @PostMapping("/missions")
            public ApiResponse<PageResDTO<MissionResDTO.MissionDetailDTO>> getMissions(
                    @RequestBody MissionReqDTO.GetMyMissionsReqDTO request,
                    @RequestParam String status,
                    @RequestParam(defaultValue = "0") Integer page,
                    @RequestParam(defaultValue = "3") Integer size
            ) {
                Long memberId = request.getMemberId();
                
                PageResDTO<MissionResDTO.MissionDetailDTO> resultDTO = missionService.getMyMissions(memberId, status, page, size);
        
                return ApiResponse.onSuccess(
                        MissionSuccessCode.OK,
                        resultDTO
                );
            }
        ```

    - 스웨거 실행 사진

      ![image.png](/week07/mission/mission_07_(1).png)

      ![image.png](/week07/mission/mission_07_(2).png)

---
   3. 내가 생성한 리뷰들 조회하기 (커서 기반 페이지네이션으로 응답하기, 사진 부분 제외, ID 순, 별점 순 모두 구현하기)
       - 회원 중 나의 리뷰 목록을 조회한다는 의미로 API 엔드포인트를 `GET /members/me/reviews`로 설계했다.
           - `GET /reviews/me` -> REST API에서 `/컬렉션/{id}` 구조는 보통 특정 단건을 조회
           - `GET /members/reviews` -> 모든 회원의 모든 리뷰를 가져오는 건지, 내 리뷰를 가져오는 건지 모호함
       - 별점 순으로 정렬할 때는 중복 별점이 많으므로 마지막 별점과 마지막 ID 2개의 데이터가 커서로 필요하다.
         워크북 방식처럼 단일 커서를 쓰면 문자열을 쪼개는 과정이 필요하기 때문에, lastReviewId와 lastRating로 분리하였다.
       - API 명세서
           - API 개요
               - 내가 작성한 리뷰 목록을 조회하는 API입니다.
               - 커서 기반 페이지네이션을 사용합니다.
               - Query String을 통해 최신순(ID 순) 또는 별점순으로 정렬할 수 있습니다.
           - API Endpoint
               - `GET /members/me/reviews`
           - Request Header

               ```json
               Authorization: Bearer {accessToken}
               ```

           - Query String

               | **Key** | **Type** | **Required** | **Description** |
               | --- | --- | --- | --- |
               | sort | String | X | 정렬 기준 (`LATEST`: 최신순/ID순, `RATING`: 별점순). 기본값 `LATEST` |
               | lastReviewId | Long | X | 이전 페이지의 마지막 리뷰 ID (첫 페이지 조회 시 생략) |
               | lastRating | Float | X | 이전 페이지의 마지막 별점 (별점순 정렬 시 중복 방지를 위해 `lastReviewId`와 함께 사용, 최신순 정렬이거나 첫 페이지 조회 시 생략) |
               | size | Integer | X | 페이지당 개수, 기본값 3 |
           - Success Response
            
               ```json
               {
                 "isSuccess": true,
                 "code": "REVIEW200_1",
                 "message": "내가 작성한 리뷰 목록 조회에 성공했습니다.",
                 "result": {
                   "data": [
                     {
                       "reviewId": 85,
                       "storeId": 31,
                       "storeName": "반이학생마라탕마라반",
                       "nickname": "닉네임1234",
                       "rating": 4.5,
                       "content": "음 너무 맛있어요 포인트도 얻고 맛있는 맛집도 알게 된 것 같아 너무나도 행복한 식사였답니다. 다음에 또 올게요!!",
                       "createdAt": "2022-05-14T18:30:00",
                       "ownerReply": "감사합니다.",
                       "ownerReplyCreatedAt": "2022-05-15T10:00:00"
                     },
                     {
                       "reviewId": 82,
                       "storeId": 12,
                       "storeName": "컴포즈커피",
                       "nickname": "닉네임1234",
                       "rating": 4.5,
                       "content": "커피가 저렴해서 좋아요.",
                       "createdAt": "2022-05-09T20:15:00",
                       "ownerReply": null,
                       "ownerReplyCreatedAt": null
                     }
                   ],
                   "pageInfo": {
                     "hasNext": true,
                     "nextCursorId": 82,
                     "nextCursorRating": 4.5
                   }
                 }
               }
               ```
            
       - 기존 오프셋 기반의 공통 페이징 응답 객체 `PageResDTO`은 커서 기반 페이지네이션에서는 전체 개수 등 필요한 메타데이터 구조가 다르기 때문에 사용할 수 없어, 리뷰 도메인 커서 페이징 응답 객체를 따로 생성했다.
       ```java
           public class ReviewResDTO {
    
           // 내 리뷰 목록
           @Builder
           public record ReviewCursorPageDTO(
                   List<ReviewDetailDTO> data,
                   ReviewCursorInfoDTO pageInfo
           ) {}
    
           @Builder
           public record ReviewDetailDTO(
                   Long reviewId,
                   Long storeId,
                   String storeName,
                   String nickname,
                   Float rating,
                   String content,
                   LocalDateTime createdAt,
                   String ownerReply,
                   LocalDateTime ownerReplyCreatedAt
           ) {}
    
           // 커서 페이징 메타데이터
           @Builder
           public record ReviewCursorInfoDTO(
                   Boolean hasNext,
                   Long nextCursorId,
                   Float nextCursorRating
           ) {}
        }
        ```
      
   - 스웨거 실행 사진
        - 최신순
            
            ![첫 페이지](/week07/mission/mission_07_(3).png)
            
            첫 페이지
            
            ![lastReviewId=12 요청](/week07/mission/mission_07_(4).png)
            
            lastReviewId=12 요청 -> lastReviewId=11부터 응답
            
        - 별점순
            
            ![첫 페이지](/week07/mission/mission_07_(5).png)
            
            첫 페이지
            
            ![lastReviewId=5, lastRating=5 요청](/week07/mission/mission_07_(6).png)
            
            lastReviewId=5, lastRating=5 요청

4. Request Body가 있는 API에 검증 어노테이션 붙혀 검증하기
    - 리뷰 작성 API에 검증 어노테이션을 추가했다.
        - 요청 DTO의 storeId, rating에 검증 어노테이션을 추가했다.

        ```java
        public class ReviewReqDTO {
        
            // 리뷰 작성
            public record CreateReviewDTO(
        
                    @NotNull(message = "가게 ID는 필수입니다.")
                    Long storeId,
                    @NotNull(message = "별점은 필수입니다.")
                    @DecimalMin(value = "0.0", message = "별점은 0.0 이상이어야 합니다.")
                    @DecimalMax(value = "5.0", message = "별점은 5.0 이하이어야 합니다.")
                    Float rating,
                    String content,
                    List<Long> imageIds
            ) {}
        }
        ```

    - 스웨거 실행 사진

      ![image.png](/week07/mission/mission_07_(7).png)