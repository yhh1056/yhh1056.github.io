---
title:  "[JPA] @Modifying을 사용한 벌크 연산시 주의 사항"
excerpt: ""
toc: true
toc_sticky: true

categories:
  - contents
last_modified_at: 2022-08-31TO00:40:00+09:00
---

### 미팅 완료 버튼을 누르지 않은 사용자들을 위한 스케줄링

땡쿠 서비스는 쿠폰을 통해 예약이 승인되면 미팅이라는 티켓을 생성합니다.
미팅은 언제든지 사용완료를 할 수 있는데요. 사용자라면 보통 미팅을 한 후 미팅 완료 버튼을 누를겁니다. 하지만 까먹기 쉬운 부분이기 때문에 완료 버튼을 누르지 않을 수도 있겠죠?

<img width="315" alt="스크린샷 2022-08-31 오후 3 13 41" src="https://user-images.githubusercontent.com/58363663/187607142-7efbf64f-dd3c-4fb6-8db5-ec32d624e118.png">

그렇게 되면 쿠폰의 상태가 계속 `사용 완료`된 상태가 아니라 `예약된` 상태로 머무르게 됩니다. 물론 예약된 쿠폰을 조회하는 경우 문제가 발생할겁니다.

그래서 땡쿠팀은 매일 스케줄 작업을 통해 미팅 날짜가 하루가 지나게 된다면 `완료 처리`를 해주는 작업을 실행하고 있습니다.

```java
@Component
@RequiredArgsConstructor
public class MeetingScheduleTask {

  public static final Long DAY = 1L;

  private final MeetingService meetingService;

  @Scheduled(cron = "0 0 2 * * *")
  public void executeCompleteMeeting() {
    meetingService.complete(LocalDate.now().minusDays(DAY));
  }
}
```

### @Modifying
`@Query`어노테이션으로 작성된 메서드에 insert, update, delete 연산을 하게되는 경우 `@Modifying` 어노테이션을 함께 사용해야 합니다.

```java
@Modifying(clearAutomatically = true)
@Query("UPDATE Meeting m SET m.meetingStatus = :status WHERE m.id IN (:meetingIds)")
void updateMeetingStatus(@Param("status") MeetingStatus status, @Param("meetingIds") List<Long> meetingIds);

@Modifying(clearAutomatically = true)
@Query("UPDATE Coupon c SET c.couponStatus = :status WHERE c.id IN (:couponIds)")
void updateCouponStatus(@Param("status") CouponStatus status, @Param("couponIds") List<Long> couponIds);
```


하지만 문제가 있습니다. @Query를 사용하는 경우 영속성 컨텍스트를 거쳐 select문을 실행하게 되는데 @Modifying을 사용하게 된다면 영속성 컨텍스트를 거치지 않고 쿼리를 직접 날립니다.

그러면 어떤 문제가 발생하게 될까요?

1. 미팅을 저장한다. → 영속성 컨텍스트에 해당 미팅이 존재
2. 미팅을 수정한다. → 영속성 컨텍스트를 거치지 않고 쿼리 실행
    - 데이터 베이스는 update쿼리가 작동하여 데이터가 변경
    - 영속성 컨텍스트에는 1번에서 저장한 미팅이 그대로 존재
3. 수정된 미팅을 기대하고 미팅을 조회 → 1번에서 저장된 미팅이 반환


이렇게 데이터 정합성이 맞지 않게 되고 테스트 코드가 실패하게 될겁니다. 그래서 @Modifying 어노테이션에는 `clearAutomatically`를 true로 설정합니다..

영속성 컨텍스트에는 1번이 저장되어 조회할 경우 계속 변경되지 않은 미팅이 조회될 것이므로 영속성 컨텍스트를 비우고 난 뒤 조회를 하게되면 변경된 미팅이 조회될겁니다.

```java
@Modifying(clearAutomatically = true)
@Query(value = "UPDATE Meeting m SET m.status = ? WHERE m.date = ? AND m.status = ?", nativeQuery = true)
void expire(String updatedStatus, LocalDate date, String originStatus);
```

### 결론

`@Modifying` 어노테이션이 영속성 컨텍스트를 거치지 않는다는 점을 모르고 있었고 몇시간 삽질을 하게 되었습니다.

**clearAutomatically** 을 사용하게 되면 메서드를 실행할 경우 컨텍스트가 비워지기 때문에 더티 체킹이 필요한 곳에 사용하면 안되겠죠?