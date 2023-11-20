# 섹션 2. API 개발 고급 - 준비

## API 고급 소개 
장애의 90% 정도는 조회에서 발생한다. 성능 관련 문제나 Lazy 로딩 관련 문제 등 조회 관련 API에 대해 다룬다. 

## 조회용 샘플 데이터 입력

단순 예제이므로 InitDb 클래스를 생성해 @Component 어노테이션을 추가한다. @Component 어노테이션을 추가하면 자동으로 component scan의 대상이 된다.

```java
package jpabook.jpashop;

import jakarta.annotation.PostConstruct;
import jakarta.persistence.EntityManager;
import jpabook.jpashop.domain.*;
import jpabook.jpashop.domain.item.Book;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

@Component
@RequiredArgsConstructor
public class InitDb {

    private final InitService initService;
    
    @PostConstruct
    public void init() {
        initService.dbInit1();
    }
    
    @Component
    @Transactional
    @RequiredArgsConstructor
 static class InitService {
     
        private final EntityManager em;
        public void dbInit1() {
            Member member = new Member();
            member.setName("userA");
            member.setAddress(new Address("서울", "1", "1111"));
            em.persist(member);

            Book book1 = new Book();
            book1.setName("JPA1 BOOK");
            book1.setPrice(10000);
            book1.setStockQuantity(100);
            em.persist(book1);
            
            Book book2 = new Book();
            book2.setName("JPA2 BOOK");
            book2.setPrice(20000);
            book2.setStockQuantity(100);
            em.persist(book2);

            OrderItem orderItem1 = OrderItem.createOrderItem(book1, 10000, 1);
            OrderItem orderItem2 = OrderItem.createOrderItem(book2, 20000, 2);
            
            Delivery delivery = new Delivery();
            delivery.setAddress(member.getAddress());

            Order order = Order.createOrder(member, delivery, orderItem1, orderItem2);
            
            em.persist(order);
        }
 }   
}
```

@PostConstruct는 스프링 빈이 엮이고 나서 마지막에 스프링이 호출해준다. 참고로 init과 dbInit1은 스프링 라이프사이클 상 별도로 분리해주어야 한다.


## OrderList.html 수정

추가적으로, 기존 예제는 주문목록 조회 html부분에서 회원의 주문상품이 여러 개인 경우 제일 처음 주문한 주문상품에 대해서만 출력하고 있다.
편의를 위해 html의 테이블 부분을 다음과 같이 수정해 주었다. 강의 내용과는 관계 없다!

```html
<table class="table table-striped">
            <thead>
            <tr>
                <th>#</th>
                <th>회원명</th>
                <th>상품 이름</th>
                <th>상품 주문가격</th>
                <th>상품 주문수량</th>
                <th>상태</th>
                <th>일시</th>
                <th></th>
            </tr>
            </thead>
            <tbody>
            <tr th:each="item : ${orders}">
                <td th:text="${item.id}"></td>
                <td th:text="${item.member.name}"></td>
                <td>
                    <div th:each="orderItem : ${item.orderItems}">
                        <p th:text="${orderItem.item.name}"></p>
                    </div>
                </td>
                <td>
                    <div th:each="orderItem : ${item.orderItems}">
                        <p th:text="${orderItem.orderPrice}"></p>
                    </div>
                </td>
                <td>
                    <div th:each="orderItem : ${item.orderItems}">
                        <p th:text="${orderItem.count}"></p>
                    </div>
                </td>
                <td th:text="${item.status}"></td>
                <td th:text="${item.orderDate}"></td>
                <td>
                    <a th:if="${item.status.name() == 'ORDER'}" href="#" th:href="'javascript:cancel('+${item.id}+')'"
                       class="btn btn-danger">CANCEL</a>
                </td>
            </tr>

            </tbody>
        </table>
```