# API 개발 고급 - 지연 로딩과 조회 성능 최적화

xToOne 관계에서 지연 로딩때문에 발생하는 성능 문제를 단계적으로 해결해본다.

## 간단한 주문 조회 V1: 엔티티를 직접 노출 


```java
@GetMapping("/api/v1/simple-orders")
public List<Order> ordersV1() {
    List<Order> all = orderRepository.findAllByString(new OrderSearch());
    return all;
}
```

이런 식으로 코드를 작성하면, 무한 루프에 빠지게 된다.
이런 양방향 관계에서는 한쪽을 @JsonIgnore를 해주어야 해결할 수 있다.

그러나 @JsonIgnore를 추가해줘도 에러가 발생한다. 
Order의 Member 페치를 살펴보면 지연 로딩으로 되어있다.  
이 말은 즉 DB에서 가져올 때 Member에 프록시 ByteBuddy를 대신 넣어두고, Member 객체에 값을 꺼내거나 손대는 경우 Member의 값을 가져오게 된다.
순수한 Member 객체가 아니므로 오류가 다시 발생하게 되는 것이다.

이를 해결하기 위해 Hibernate5Module이 필요하다.

build.gradle에 `implementation 'com.fasterxml.jackson.datatype:jackson-datatype-hibernate5'`를 추가하고,
이를 Bean으로 등록한다.

(참고로 내 환경에서는 `implementation 'com.fasterxml.jackson.datatype:jackson-datatype-hibernate5-jakarta`를 추가해줘야 한다.)

이렇게 해주면 지연 로딩한 값을 Null로 받아올 수 있고, ForceLazyLoading 옵션을 이용해 강제로 로딩할 수도 있다. 

그러나 앞서 섹션 2에서도 언급했듯, 엔티티를 외부에 노출하는 것은 매우 위험하다!!!
또한, 성능 상으로도 문제가 있다. 원하지 않는 정보를 API 스펙에 노출하는 것뿐만 아니라 다 가져와야 한다는 엄청난 성능 낭비 문제가 있는 것이다.

그렇다고 LAZY를 EAGER로 바꾸는 것은 N+1의 문제를 발생시킬 수 있고, 필요하지 않은 정보를 강제로 가져와야 하므로 성능 최적화에 어려움이 있으므로 지양하자.

## 간단한 주문 조회 V2: 엔티티를 DTO로 변환

필요한 정보만 담아 반환하는 DTO를 만든다. 

```java
@GetMapping("/api/v2/simple-orders")
public List<SimpleOrderDto> ordersV2() {
    return orderRepository.findAllByCriteria(new OrderSearch()).stream()
            .map(SimpleOrderDto::new)
            .collect(Collectors.toList());

}

@Data
static class SimpleOrderDto {
    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address;

    public SimpleOrderDto(Order order) {
        orderId = order.getId();
        name = order.getMember().getName();
        orderDate = order.getOrderDate();
        orderStatus = order.getStatus();
        address = order.getDelivery().getAddress();
    }
}
```

그러나 V1과 V2 모두 Lazy 로딩으로 인한 데이터베이스 쿼리가 많이 발생한다는 문제가 있다. 
`name = order.getMember().getName();` 에서 Lazy가 초기화된다. 즉, 영속성 컨텍스트가 memberID를 찾아보고 없으면 DB 쿼리를 날려 데리터를 끌고 온다.

sql 1회 실행으로 결과 row가 2개 나온다. 이후 loop를 돌며 첫번째 바퀴에서 Member와 Delivery 쿼리를 하나씩 날리고, 두번쨰 바퀴에서 같은 방식으로 정보를 가져온다.  
결과적으로 총 5번 쿼리가 발생했는데, orders에서 회원 n개를 가져오는 첫번째 쿼리 한 개로 **1(Order 조회) + N(회원) + N(배송)** 번의 쿼리가 발생한 것이다.  

그렇다고 Lazy를 Eager로 변경하면 성능이 개선되지 않을 뿐더러 예상치 못한 쿼리가 발생한다.

참고로, 현재 초기 데이터는 멤버가 2명인 경우지만 멤버 1명이 2번의 주문을 한 경우에는 영속성 컨텍스트에 있는 값을 가져올 수 있으므로 쿼리가 한번 줄게 된다. 
그러나 위처럼 최악의 경우를 산정하는 게 맞다!

## 간단한 주문 조회 V3: 엔티티를 DTO로 변환 - 페치 조인 최적화

위의 N+1 문제를 페치 조인으로 최적화해보자.

먼저, OrderRepository에 다음과 같은 메서드를 추가한다.

```java
public List<Order> findAllWithMemberDelivery() {
    return em.createQuery(
            "select o from Order o" +
                    " join fetch o.member m" +
                    " join fetch  o.delivery d", Order.class
            ).getResultList();
}
```

즉, 한번의 쿼리로 Order, Member, Delivery를 조인한 뒤 한번에 가져오는 것이다. 
이 경우에는 LAZY더라도 프록시가 아닌 실제 값을 가져오게 된다. (지연로딩 X)

```java
@GetMapping("/api/v3/simple-orders")
public List<SimpleOrderDto> ordersV3() {
    List<Order> orders = orderRepository.findAllWithMemberDelivery();

    return orders.stream()
            .map(SimpleOrderDto::new)
            .collect(Collectors.toList());
}
```
사실상 v2와 v3는 완전히 같은 로직이나, v3의 경우 페치 조인으로 쿼리가 한번 나가게 된다!

## 간단한 주문 조회 V4: JPA에서 DTO로 바로 조회

여기서 더 최적화를 해보자!

기존의 방식들은 엔티티에서 조회한 후 DTO로 넘겨주었는데, 이런 과정 없이 바로 DTO로 데이터를 가져와본다.

OrderSimpleQueryDto 클래스를 별도로 생성해준다.

```java
package jpabook.jpashop.repository;

import jpabook.jpashop.domain.Address;
import jpabook.jpashop.domain.Order;
import jpabook.jpashop.domain.OrderStatus;
import lombok.Data;

import java.time.LocalDateTime;

@Data
public class OrderSimpleQueryDto {
    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address;

    public OrderSimpleQueryDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address) {
        this.orderId = orderId;
        this.name = name;
        this.orderDate = orderDate;
        this.orderStatus = orderStatus;
        this.address = address;
    }
}
```

이후 OrderRepository에서 다음과 같은 메서드를 추가한다.
참고로 이때는 단순히 select o가 아닌 new 명령어를 사용해 다음과 같은 식으로 값을 직접 넘겨줘야 식별자를 넘겨주는 것이 아니라 원하는 값을 넘겨주도록 한다.

```java
public List<OrderSimpleQueryDto> findOrderDtos() {
    return em.createQuery(
            "select new jpabook.jpashop.repository.OrderSimpleQueryDto(o.id, m.name, o.orderDate, o.status, d.address)  " +
                    " from Order o" +
                    " join o.member m" +
                    " join o.delivery d", OrderSimpleQueryDto.class)
            .getResultList();
}
```

이후 컨트롤러에서 v4 메서드를 추가해준다.

```java
@GetMapping("/api/v4/simple-orders")
public List<OrderSimpleQueryDto> ordersV4() {
    return orderRepository.findOrderDtos();
}
```

v3과 v4의 쿼리를 비교해보자.

```sql
select
        o1_0.order_id,
        d1_0.delivery_id,
        d1_0.city,
        d1_0.street,
        d1_0.zipcode,
        d1_0.status,
        m1_0.member_id,
        m1_0.city,
        m1_0.street,
        m1_0.zipcode,
        m1_0.name,
        o1_0.order_date,
        o1_0.status 
    from
        orders o1_0 
    join
        member m1_0 
            on m1_0.member_id=o1_0.member_id 
    join
        delivery d1_0 
            on d1_0.delivery_id=o1_0.delivery_id
```

앞서 언급한 대로 v3은 페치조인으로 order, member, delivery가 모두 조인되어 출력된다.

```sql
select
        o1_0.order_id,
        m1_0.name,
        o1_0.order_date,
        o1_0.status,
        d1_0.city,
        d1_0.street,
        d1_0.zipcode 
    from
        orders o1_0 
    join
        member m1_0 
            on m1_0.member_id=o1_0.member_id 
    join
        delivery d1_0 
            on d1_0.delivery_id=o1_0.delivery_id

```

반면 v4의 경우 원하는 데이터만 select 해서 가져오고 있다.

**그렇다면 v4가 더 좋은 것일까? 이는 trade-off가 있다.**
v4는 화면에는 최적화되어 있으나 리포지토리 재사용성은 v3에 비해 떨어진다. 로직을 더이상 재활용할 수 없으며 비교적 지저분하다는 단점이 있다. 어떻게 보면 v4는 리포지토리가 화면에 의존하게 만들고 있는 것이다. 
한편, v3는 엔티티로 조회했기 때문에 데이터를 변경할 수 있으나, v4의 경우 Dto이기 때문에 데이터를 변경할 수 없고, v4가 조금 더 최적화할 수 있다.

사실상 v3과 v4 정도는 전체적인 애플리케이션 관점에서 성능상 크게 차이가 나진 않으나 데이터 사이즈가 큰 경우에는 고민해 볼 법하다.

강사님의 추천은 쿼리용, 성능 최적화용 디렉토리를 별도로 생성하는 것이다. 화면에 dependency 하나 쿼리가 복잡한 경우, 리포지토리에 있으면 용도가 애매해지고 유지보수가 어렵기 때문이다.

**정리해보면,** 

엔티티를 DTO로 변환하거나 DTO로 즉시 반환하는 것은 각자 장단점이 있다.

우선 엔티티를 DTO로 변환하는 방법을 선택하고, 필요 시 페치 조인으로 성능 최적화하면 대부분의 성능 이슈가 해결된다.  
그러나 이래도 안되는 경우 DTO로 직접 조회하는 방법을 사용하고, 최후의 경우 JPA의 네이티브 SQL이나 JDBC Template을 사용해 SQL을 직접 사용한다!