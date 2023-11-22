# API 개발 고급 - 컬렉션 조회 최적화

컬렉션 조회는 일대다 조회이므로 전체 로그가 다만큼 늘어나 최적화가 아렵게 된다. 
섹션 3과 같이 단계적으로 최적화해본다.

Order의 경우 OrderItem이 리스트로 들어가 있고, OrderItem은 Item과 ManyToOne 관계이다. 
주문 내역과 주문 상품명을 어떻게 출력하는지 만들어보자.

## 주문 조회 V1: 엔티티 직접 노출

```java
@GetMapping("/api/v1/orders")
public List<Order> ordersV1() {
    List<Order> all = orderRepository.findAllByString(new OrderSearch());
    for (Order order : all) {
        order.getMember().getName();
        order.getDelivery().getAddress();
        List<OrderItem> orderItems = order.getOrderItems();
        orderItems.forEach(o-> o.getItem().getName());
    }
    return all;
}
```
앞선 섹션 3와 마찬가지로 프록시를 강제 초기화해주며 양방향 관계는 @JsonIgnore를 해준다.

이 방법은 엔티티를 직접 노출하는 것이므로 지양하자.

## 주문 조회 V2: 엔티티를 DTO로 변환

```java
@GetMapping("/api/v2/orders")
public List<OrderDto> ordersV2() {
    List<Order> orders = orderRepository.findAllByString(new OrderSearch());
    List<OrderDto> collect = orders.stream()
            .map(o -> new OrderDto(o))
            .collect(Collectors.toList());

    return collect;
}

@Data
static class OrderDto {

    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address;
    private List<OrderItem> orderItems;

    public OrderDto(Order order) {
        orderId = order.getId();
        name = order.getMember().getName();
        orderDate = order.getOrderDate();
        orderStatus = order.getStatus();
        address = order.getDelivery().getAddress();
        orderItems = order.getOrderItems();
    }
}
```

v2를 실행해보면 orderItems는 엔티티이기 때문에 나오지 않는다.   
이때 생성자에서 orderItems 위에 `order.getOrderItems().forEach(o -> o.getItem().getName());` 로 초기화해주면 orderItems가 나오게 된다.

여기서 문제점은 DTO 안에 엔티티가 있다는 것이다. DTO로 반환할 때는 내부에 엔티티가 존재해서는 안되며 래핑하는 것도 안 된다.   
엔티티가 외부에 노출되면 안된다는 뜻은 지금과 같이 단순히 DTO로 감싸야 된다는 뜻이 아니라 **엔티티에 대한 의존을 완전히 끊어야 한다** 는 뜻이다.

즉, orderItem 조차도 DTO여야 한다! (참고로 Address 같은 value object는 노출되어도 괜찮다!)

```java
@Getter
static class OrderItemDto {

    private String itemName;
    private int orderPrice;
    private int count;

    public OrderItemDto(OrderItem orderItem) {
        itemName = orderItem.getItem().getName();
        orderPrice = orderItem.getOrderPrice();
        count = orderItem.getCount();
    }
}
```

OrderItemDto 클래스를 생성한 뒤, orderItems를 OrderItemDto와 매핑한다.

이때 쿼리가 어마무시하게 많이 발생되는 것을 확인할 수 있다.

## 주문 조회 V3: 엔티티를 DTO로 변환 - 페치 조인 최적화

방금 전 예제를 페치 조인으로 최적화해보자.

```java
public List<Order> findAllWithItem() {
    return em.createQuery(
            "select o from Order o" +
                    " join fetch o.member m" +
                    " join fetch o.delivery d" +
                    " join fetch o.orderItems oi" +
                    " join fetch oi.item i", Order.class)
    .getResultList();
}
```

현재 2개인 order와 4개인 orderItem을 조인하면 order는 총 4개가 된다. 
이는 일대다 조인에서 orderItem이 많으면 많을수록 데이터의 양이 증가될 것을 암시한다.

이때, distinct 키워드를 추가해주면 DB에도 distinct가 적용된 쿼리를 날려주고, 엔티티가 중복인 경우 중복을 제거해 컬렉션에 담아준다.

참고로 내가 사용하고 있는 버전에서는 페치 조인 시 자동으로 distinct가 적용되어 자동으로 중복이 제거된다고 한다.
[참고 링크](https://www.inflearn.com/questions/807577/spring-boot-3-x-distinct-%EA%B4%80%EB%A0%A8)

v3과 같이 페치 조인을 사용하면 사실상 같았던 코드지만 v2에서는 여러번 나가던 쿼리가 한번으로 줄어들게 된다!

그러나, 컬렉션 페치 조인은 **페이징이 불가** 하다는 치명적인 단점이 있다.

우리가 기대하는 order는 2개이지만, DB 상에서 order는 4개이다. 
일대다 조인을 한 순간 우리가 원하는 order와는 사이즈가 달라지기 때문에 페이징 자체가 불가능하다.

일대다 페치 조인이 들어가는 순간 하이버네이트는 경고를 발생하고 메모리에서 처리한다. 이는 큰 이슈를 발생할 수 있는 매우 위험한 방식이다.

또한, 컬렉션 페치 조인은 1개만 사용할 수 있다. 

자세한 건 자바 ORM 표준 JPA 프로그래밍을 참조하자.   
이 부분은 [내 블로그](https://yeondori.github.io/posts/jpa-basic-11/#%EC%BB%AC%EB%A0%89%EC%85%98-%ED%8E%98%EC%B9%98-%EC%A1%B0%EC%9D%B8)에 정리되어 있다.

## 주문 조회 V3.1: 엔티티를 DTO로 변환 - 페이징과 한계 돌파

그렇다면 **페이징과 한계를 어떻게 돌파** 해야 할까?

우리는 일대다에서 일을 기준으로 페이징하고 싶다. 대부분의 페이징 + 컬렉션 엔티티 조회 문제는 다음의 방법으로 해결할 수 있다.

> 1. ToOne 관계를 모두 페치 조인한다. ToOne 관계에서는 데이터 Row 수가 증가하지 않는다. (데이터가 없는 경우 제외)
> 2. 컬렉션은 지연 로딩으로 조회한다.
> 3. 지연 로딩 성능 최적화를 위해 hibernate.default_batch_fetch_size, @BatchSzie를 적용한다.

**application.yml 파일에 `default_batch_fetch_size: 100`를 추가한다.** 

이때 쿼리를 살펴보면, order의 아이디를 가지고 컬렉션과 관련된 것을 in 쿼리로 한번에 가져오게 된다.   
여기서 fetch_size 100은 **in 쿼리의 개수**를 의미한다.만약 size가 10이고 데이터가 100개 있다면 in 쿼리가 10번 발생하는 것이다.   
이러한 방식은 1 + N 쿼리 수가 1 + 1로 최적화되며, 조인보다 DB 전송량이 최적화되고 페이징이 가능하다는 장점이 있다.

이전 방식에 비해 쿼리는 더 나갔지만, 불필요한 중복 데이터 없이 필요한 데이터를 정확하게 전송받아 최적화할 수 있으므로 전송량 자체가 줄게 된다.
이는 글로벌한 경우로, 보다 디테일하게 적용하고 싶은 경우는 **@BatchSize 어노테이션** 을 이용한다.

참고로 default_batch_fetch_size 는 100~1000을 권장한다. 
성능상 1000으로 설정하는 것이 가장 좋으나, DB와 애플리케이션에 순간적으로 부하가 증가할 수 있으므로 부하를 어디까지 견딜 수 있는가를 고민해보며 결정하면 된다.

## 주문 조회 V4: JPA에서 DTO 직접 조회


```java
import jakarta.persistence.EntityManager;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.stream.Collectors;

@Repository
@RequiredArgsConstructor
public class OrderQueryRepository {

    private final EntityManager em;

    public List<OrderQueryDto> findOrderQueryDtos() { //루트 조회(toOne 코드를 모두 한번에 조회)
        List<OrderQueryDto> result = findOrders();
        result.forEach(o -> {
            List<OrderItemQueryDto> orderItems = findOrderItems(o.getOrderId());
            o.setOrderItems(orderItems);
        });
        return result;
    }

    /**
     * 1:N 관계(컬렉션)를 제외한 나머지를 한번에 조회
     */
    private List<OrderQueryDto> findOrders() {
        return em.createQuery(
                        "select new jpabook.jpashop.repository.order.query.OrderQueryDto(o.id, m.name, o.orderDate, o.status, d.address)" +
                                " from Order o" +
                                " join o.member m" +
                                " join o.delivery d", OrderQueryDto.class)
                .getResultList();
    }

    /**
     * 1:N 관계인 orderItems 조회
     */
    private List<OrderItemQueryDto> findOrderItems(Long orderId) {
        return em.createQuery(
                        "select new jpabook.jpashop.repository.order.query.OrderItemQueryDto(oi.order.id, i.name, oi.orderPrice, oi.count)" +
                                " from OrderItem oi" +
                                " join oi.item i" +
                                " where oi.order.id = : orderId", OrderItemQueryDto.class)
                .setParameter("orderId", orderId)
                .getResultList();
    }
}
```
디렉토리를 별도로 생성해 Dto와 Repository를 생성한다.

row 수가 증가하지 않는 ToOne 관계를 조인으로 먼저 조회하고, 최적화가 어려운 ToMany 관계는 별도로 처리한다.

이렇게 하면 쿼리는 루트 1번 컬렉션 N번 실행한다. 
즉 N + 1 문제가 발생한다. 

## 주문 조회 V5: JPA에서 DTO 직접 조회 - 컬렉션 조회 최적화

```java
public List<OrderQueryDto> findAllByDto_optimization() {
    List<OrderQueryDto> result = findOrders();

    List<Long> orderIds = result.stream()
            .map(OrderQueryDto::getOrderId)
            .collect(Collectors.toList());

    List<OrderItemQueryDto> orderItems = em.createQuery(
                    "select new jpabook.jpashop.repository.order.query.OrderItemQueryDto(oi.order.id, i.name, oi.orderPrice, oi.count)" +
                            " from OrderItem oi" +
                            " join oi.item i" +
                            " where oi.order.id in :orderIds", OrderItemQueryDto.class)
            .setParameter("orderIds", orderIds)
            .getResultList();

    Map<Long, List<OrderItemQueryDto>> orderItemMap = orderItems.stream()
            .collect(Collectors.groupingBy(OrderItemQueryDto::getOrderId));
    
    result.forEach(o -> o.setOrderItems(orderItemMap.get(o.getOrderId())));

    return result;
}
```

orderId를 in 쿼리로 바꾸고, orderItems을 Map으로 바꿔 매칭 성능을 향상시킨다.
이렇게 하면 쿼리를 루트 한번, 컬렉션 한번 총 2번으로 최적화할 수 있다.

이러한 방식은 페치 조인에 비해 데이터를 select 하는 양이 줄어든다.

## 주문 조회 V6: JPA에서 DTO 직접 조회, 플랫 데이터 최적화

쿼리를 한번으로 줄여보자!

OrderFlatDto를 생성하고 모든 값을 flat하게 받아온다. 

즉, 다음의 메서드를 생성한다. 

```java
public List<OrderFlatDto> findAllByDto_flat() {
    return em.createQuery(
            "select new jpabook.jpashop.repository.order.query.OrderFlatDto(o.id, m.name, o.orderDate, o.status, d.address, i.name, oi.orderPrice, oi.count)" +
                    " from Order o" +
                    " join o.member m" +
                    " join o.delivery d" +
                    " join o.orderItems oi" +
                    " join oi.item i", OrderFlatDto.class)
            .getResultList();
}
```

그러나 이때 결과를 살펴보면 데이터가 중복으로 출력된다.

가장 큰 장점은 하나의 쿼리만 나오고, 페이징 기능은 가능하나 우리가 원하는 order 기준의 페이징은 불가하다.  
또한, API 스펙을 v5처럼 OrderQueryDto로 맞추고 싶다면, 다음과 같은 코드를 추가해준다.


```java
@GetMapping("/api/v6/orders")
public List<OrderQueryDto> ordersV6() {
    List<OrderFlatDto> flats = orderQueryRepository.findAllByDto_flat();

    return flats.stream()
            .collect(groupingBy(o -> new OrderQueryDto(o.getOrderId(), o.getName(), o.getOrderDate(), o.getOrderStatus(), o.getAddress()),
                    mapping(o -> new OrderItemQueryDto(o.getOrderId(), o.getItemName(), o.getOrderPrice(), o.getCount()), toList())
            )).entrySet().stream()
            .map(e -> new OrderQueryDto(e.getKey().getOrderId(), e.getKey().getName(), e.getKey().getOrderDate(), e.getKey().getOrderStatus(), e.getKey().getAddress(), e.getValue()))
            .collect(toList());
}
```

참고로 OrderQueryDto에 `@EqualsAndHashCode(of = "orderId")`를 추가해 주어야 orderId를 기준으로 grouping 한가.

위의 방식은 쿼리가 한번이지만 데이터가 엄청 크다든가 상황에 따라 V5보다 느릴 수도 있다.
애플리케이션에서 추가 작업이 크고 Order를 기준으로 페이징하는 것이 불가능하다는 단점이 있다.

## API 개발 고급 정리 

정리해보면, 엔티티 조회와 DTO 직접 조회가 있다.

#### 엔티티 조회
엔티티 조회후 그대로 반환(V1)하거나 DTO로 변환해 반환(V2)하는 방법이 있으며 후자를 권장한다.  

그러나 이 두가지 경우 여러 테이블이 엮인 경우 성능이 안나올 수 있는데, 페치 조인으로 쿼리 수를 최적화할 수 있고(V3), 이 경우 컬렉션은 페이징이 불가능하다. 
따라서 컬렉션이 아닌 ToOne 관계는 페치 조인으로 쿼리 수를 최적화하고, 컬렉션은 지연 로딩을 유지하고 hibernate.default_batch_fetch_size, @BatchSize로 최적화한다.

#### DTO 직접 조회

JPA에서 DTO를 직접 조회(V4)할 수 있으나 컬렉션 조회의 경우 일대다 관계인 컬렉션의 경우 데이터가 늘어나게 된다.
일을 기준으로 조회하고 싶은 경우 IN 절을 활용해 메모리에서 미리 조회해서 최적화(V5)할 수 있다.
또한, JOIN 결과를 그대로 조회한 후 애플리케이션에서 원하는 모양으로 직접 변환하는(V6) 플랫 데이터 최적화 방식이 있다.

### 권장 순서

1. 엔티티 조회 방식으로 우선 접근
   1. 페치 조인으로 쿼리 수를 최적화
   2. 컬렉션 최적화
      1. 페이징이 필요한 경우: hibernate.default_batch_fetch_size, @BatchSize로 최적화
      2. 페이징이 필요하지 않은 경우: 페치 조인 사용

2. 해결이 안되는 경우 DTO 조회 방식 사용
3. 해결이 안되는 경우 NativeSQL 이나 스프링 JdbcTemplate 사용

엔티티 조회 방식은 페치 조인 사용과 hibernate.default_batch_fetch_size, @BatchSize 등 코드를 거의 수정하지 않고 옵션만 약간 변경해 다양하게 성능 최적화를 시도할 수 있다.   

반면 DTO를 직접 조회하는 방식은 성능 최적화나 최적화 방식 변경 시 많은 코드를 변경해야 한다는 어려움이 있다. 
또한, 엔티티 조회와 페치 조인으로도 해결이 안되는 문제는 그만큼 트래픽이 많다는 뜻인데, 캐시 사용 등 DTO로는 해결이 되는가?에 대해서도 고민해봐야 한다. (참고로 엔티티는 직접 캐싱하면 안 된다)

개발자는 성능 최적화와 코드 복잡도 사이에서 줄타기를 해야 한다.   
엔티티 조회 방식의 장점은 JPA가 많은 부분을 최적화하기 때문에 단순한 코드를 유지하면서 성능을 최적화할 수 있다는 것이다.   
반면 DTO 조회 방식은 SQL을 직접 다루는 것과 유사하기 때문에 둘 사이에서 줄타기를 해야 한다.

DTO 조회 방식(V4, V5, V6)을 살펴보면 단순히 쿼리가 1번 실행된다고 V6이 항상 좋은 방법인 것은 아니다. 

V4는 코드가 단순해 유지보수가 쉽다. 또한 특정 주문 한건만 조회하는 경우에는 V4만으로도 성능이 잘 나온다.   
V5는 코드가 복잡하다. 대신 성능은 V4보다 훨씬 낫다. 1 + N의 쿼리를 발생시키던 것을 1 + 1로 줄일 수 있었다.  
V6는 쿼리를 1번으로 최적화되어 상당히 좋아보이나, Order 기준 페이징이 불가능하고 데이터 양이 많다. 또한 실무에서는 수백 수천건 단위로 페이징 처리가 꼭 필요하므로 이러한 경우는 선택하기 어렵다. 
또한, 데이터가 많은 경우 중복 전송이 증가하므로 V5와 비교해 성능 차이도 미비할 것이다. 