---
title:  "[JPA] 페이징 알아가기 2 (JPA)"
excerpt: ""
toc: true
toc_sticky: true

categories:
  - contents
last_modified_at: 2022-08-31TO00:40:00+09:00
---

### 들어가며

이전 [페이징 알아가기 1 (MySQL)](https://yhh1056.github.io/contents/paging-mysql/) 에서 OFFSET 방식의 문제점을 알아보았습니다.
이번엔 JPA에서 페이징을 처리할 때 어떻게 처리하는지 알아보겠습니다.

# JPA 페이징

MySQL에서 페이징을 처리하는 방법에 대해 알아보았습니다.. 이제부턴 JPA에서는 어떻게 페이징을 처리하는지, 어떤 문제가 있는지 알아보겠습니다.

### JPQL JOIN

`EntityManager`의 `setFirstResult()`, `setMaxResults()` 메서드를 사용하면 간단하게 페이징 처리를 할 수 있습니다.
```java
List<Reservation> result = entityManager.createQuery("SELECT r "
                + "FROM Reservation AS r "
                + "JOIN FETCH r.coupon AS c ", Reservation.class)
        .setFirstResult(100)
        .setMaxResults(50)
        .getResultList();
```

<img width="451" alt="스크린샷 2022-08-31 오전 1 27 30" src="https://user-images.githubusercontent.com/58363663/187490265-10ecd8b4-af5b-44ee-9247-43b06ac6d494.png">

기본적으로 `limit`과 `offset`이 사용되는 것을 볼 수 있습니다. 위는 MySQL에서 지원하는 방법이기 때문에 다른 데이터베이스를 사용할 경우 해당 데이터베이스에 지원하는 언어대로 처리가 됩니다.

<br>

### Spring Data JPA

SpringDataJpa의 경우 `Pageable` 객체를 사용한다면 더 간편하게 조회가 가능합니다.

```java
Page<Reservation> findAll(Pageable pageable);
```

```java
Page<Reservation> result = reservationRepository.findAll(PageRequest.of(0, 10));
```

<br>

🚨**하지만 문제가 있다.**

1. **count** 쿼리 발생

`Page`를 반환할 경우 `getTotalPages()` 와 `getTotalElements()` 를 위해 추가로 `count` 쿼리가 발생합니다.

<img width="371" alt="스크린샷 2022-08-31 오전 1 28 07" src="https://user-images.githubusercontent.com/58363663/187490361-d7a12fd6-f164-47c8-b816-a54d14696cfb.png">


👉 불필요한 count 쿼리를 제거하려면 `Page`를 반환받지 않고 `Slice`를 반환받으면 됩니다.

```java
Slice<Reservation> findSliceBy(Pageable pageable);
```

<img width="608" alt="스크린샷 2022-08-31 오전 1 28 28" src="https://user-images.githubusercontent.com/58363663/187490431-5de715cd-eddf-41f5-ac12-ca3b1823e1c1.png">

추가 count 쿼리가 발생하지 않게 되었네요. 총 데이터 갯수가 필요하지 않다면 Slice를 반환받아야겠네요.

2. **N + 1** 문제

`Reservation`의 경우 Coupon을 Lazy 전략으로 가지고 있기 때문에 조회해온 뒤 다시 쿠폰을 조회하기 위해 `select` 쿼리가 발생합니다. 10개의 쿠폰을 조회하기 위해 10개의 `select` 문이 추가로 발생하게 되는거죠. 이를 **N + 1 문제**라고 합니다.

<img width="608" alt="스크린샷 2022-08-31 오전 1 29 07" src="https://user-images.githubusercontent.com/58363663/187490584-8434fb25-7b98-4328-90e6-9385aa03f633.png">


XXXToOne 관계에서는 fetch join을 사용해도 문제가 되지 않습니다.

```java
@Query("select r from Reservation r join fetch r.coupon")
List<Reservation> findAllWithFetch(Pageable pageable);
```

```java
@EntityGraph(attributePaths = "coupon", type = EntityGraphType.LOAD)
Page<Reservation> findAll(Pageable pageable);
```

<img width="326" alt="스크린샷 2022-08-31 오전 1 29 37" src="https://user-images.githubusercontent.com/58363663/187490685-d2c285cc-c4bc-434e-9da7-a20fd8340ee6.png">

여전히 offset을 사용하여 조회중이네요.

<br>

XXXToMany 관계에서는 fetch join으로 데이터를 조회하는 데 문제가 발생합니다. 컬렉션을 기준으로 조회를 하기때문에 One 객체가 중복이 발생하고 jpa 입장에서는 어떤것을 기준으로 페이징처리를 해야하는지 알 수가 없어요. 따라서 데이터를 모두 조회한 뒤 어플리케이션 영역에서 페이징 작업을 처리합니다.


### NativeQuery

일반 쿼리로 해결하려 했지만 역시나 Lazy 로딩인 쿠폰때문에 N+1이 발생합니다.

```java
@Query(value = "select * "
            + "from reservation as r "
            + "join coupon as c "
            + "on r.coupon_id = c.id "
            + "where r.id > :offset "
            + "limit :limit",
            nativeQuery = true)
List<Reservation> findAll(@Param("offset") int offset, @Param("limit") int limit);
```
---


### 땡쿠는 어떻게 해야할까?

페이징을 공부할수록 아직 땡쿠팀이 페이징을 적용할 때인가? 라는 의문이 들었어요. 자신의 쿠폰이 100개가 넘는 일도 쉽지 않을 듯하고 적은 데이터에서 쿼리를 개선해도 큰 의미가 있지 않기 때문이죠.

OneToMany 관계에서 페이징 처리를 할 경우 문제가 발생할 수 있다는 것을 인지하였고 페이징을 적용해야 한다면 jdbc를 사용하는 방식을 사용하다가 추후 QueryDsl로 넘어가는 게 낫지 않을까 생각도 들었습니다.

땡쿠에서 페이징을 적용하는 시점이 다가올 때 여러 문제점들을 어떻게 해결했는지로 찾아올게요👊

---

참고 : [페이징 기능을 성능 최적화하기](https://velog.io/@now_iz/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98-%EC%B5%9C%EC%A0%81%ED%99%94%ED%95%98%EA%B8%B0)