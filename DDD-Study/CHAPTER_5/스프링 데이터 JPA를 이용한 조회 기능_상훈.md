What
---

### 1. Intro
##### CQRS
- 명령 모델과 조회 모델을 분리하는 패턴
  - 명령 모델은 상태를 변경하는 기능을 구현할 때 사용
  - 조회 모델은 데이터를 조회하는 기능을 구현할 때 사용

### 2. 검색을 위한 스펙
- 스펙
  - 검색 조건을 다양하게 조합해야 할 때 사용할 수 있는 것.
  - 애그리거트가 특정 조건을 충족하는지를 검사할 때 사용하는 인터페이스.
  ```java
    public interface Spefication<T> {
        // agg 파라미터는 검사 대상이 되는 객체.
        // 스펙을 리포지터리에 사용하면 agg는 애그리거트 루트
        // 스펙을 DAO에 사용하면 agg는 검색 결과로 리턴할 데이터 객체
        public boolean isSatisfiedBy(T agg);
    } 
    ```
  ```java
    public class OrdererSpec implements Specification<order>{
        private String ordererId;
        
        public OrdererSpec(String ordererId){
            this.ordererId = ordererId;
        }
  
        public booleanisSatisfiedBy(Order agg){
            record agg.getOrderer().getMemberId().getId().equals(ordererId);
        }
    }
    ```

### 3. 스프링 데이터 JPA를 이용한 스펙 구현
##### Specification<T>
```java
package org.springframework.data.jpa.domain;

import java.io.Serializable;
import javax.persistence.criteria.CriteriaBuilder;
import javax.persistence.criteria.CriteriaQuery;
import javax.persistence.criteria.Predicate;
import javax.persistence.criteria.Root;
import org.springframework.lang.Nullable;

public interface Specification<T> extends Serializable {
    
    @Nullable
    Predicate toPredicate(Root<T> root,
                          CriteriaQuery<?> query,
                          CriteriaBuilder cb);
}
```
##### OrdererIdSpec (entitiy : OrderSummary)
```java
public class OrdererIdSpec implements Specification<OrderSummary> {
    
    private String ordererId;

    public OrdererIdSpec(String ordererId) {
        this.ordererId = ordererId;
    }
    
    @Override
    public Predicate toPredicate(Root<OrderSummary> root,
                                 CriteriaQuery<?> query,
                                 CriteriaBuilder cb) {
        return cb.equal(root.get(OrderSummary_.orderId), ordererId);
    }
}
```

### 4. 리포지터리/DAO에서 스펙 사용하기
- 스펙을 충족하는 엔티티를 검색할 때 findAll() 사용

### 5. 스펙 조합
- 스프링 데이터 JPA가 제공하는 스펙 인터페이스는 스펙을 조합할 수 있는 두 메서드를 제공하고 있다.
```java
public interface Specification<T> extends Serializable {
    
    // 조건을 반대로 적용할 때 사용
    static <T> Specification<T> not(@Nullable Specification<T> spec) { ... }
    // null을 전달하면 아무 조건도 생성하지 않는 스펙 객체를 리턴하고
    // null이 아니면 인자로 받은 스펙 객체를 그대로 리턴함
    static <T> Specification<T> where(@Nullable Specification<T> spec) { ... }
    // 모두 충족
    default <T> Specification<T> and(@Nullable Specification<T> spec) { ... }
    // 두 스펙 중 하나 이상 충족
    default <T> Specification<T> or(@Nullable Specification<T> spec) { ... }
    
    @Nullable
    Predicate toPredicate(Root<T> root, CriteriaQuery query, CriteriaBuilder cb);
}
```

### 6. 정렬 지정하기
- 스프링 데이터 JPA는 두 가지 방법을 사용해서 정렬을 지정 할 수 있다.
  - 메서드 이름에 OrderBy를 사용해서 정렬 기준 지정
  - Sort를 인자로 전달
```java
public interface OrderSummaryDao extends Repository<OrderSummary, String> {
    
    // 정렬 기준이 하나인 경우
    // OrderBy
    // ordererId 프로퍼티 값을 기준으로 검색 조건 지정
    // number 프로퍼티 값 역순으로 정렬
    List<OrderSummary> findByOrdererIdOrderByNumberDesc(String ordererId);
    // sort
    List<OrderSummary> findByOrdererId(String ordererId, Sort sort);
    Srot sort = Sort.by("number").descendgin();
    List<OrderSummary> results = orderSummaryDAo.findByOrdererId("user1", sort);
    
    // 정렬 기준이 두 개 이상이 경우
    Sort sort1 = Sort.by("number").ascending();
    Sort sort2 = Sort.by("orderDate").descending();
    Sort sort = sort1.and(sort2);
    
    Sort sort = Sort.by("number").ascending().and(Sort.by("orderDate").descending());
}
```

### 7. 페이징 처리하기
- 스프링 데이터 JPA는 페이징 처리를 위해 Pageable 타입을 이용함.
- Sort 타입과 마찬가지로 find에 Pageable 타입 파라미터를 사용하면 페이징을 자동으로 처리해줌.
```java
public interface MemberDataDao extends Repository<MemberData, String> {
    
    // 목록
    List<MemberData> findByNameLike(String name, Pageable pageable);
    
    // 목록 뿐만 아니라 조건에 해당하는 전체 개수 및 페이징 처리에 필요한 데이터도 함께 제공
    Page<OrderSummary> findByOrderByNumberDesc(String ordererId, Pageable pageable);
        
}
//호출 예)
public class sample {

    PageRequest pageReq = PageRequest.of(1, 10);
    List<MemberData> user = memberDataDao.findByNameLike("사용자%", pageReq);
}
```
### 8. 스펙 조합을 위한 스펙 빌더 클래스
- 메서드를 사용해서 조건을 표현하고 메서드 호출 체인으로 연속된 변수 할당을 줄여 코드 가독성을 높이고 구조를 단순하게 함
##### 스펙 빌더 사용 전
```java
public class NotApplied {
    
    Specification<MemberData> spec = Specification.where(null);
    
    if (searchRequest.isOnlyNotBlocked()) {
        spec = spec.and(MemberDataSpecs.nonBlocked());
    }
    
    if (StringUtils.hasText(searchRequest.getName())) {
        spec = spec.and(MemberDataSpecs.nameLike(searchRequest.getName()));
    }
    
    List<MemberData> result = memberDataDao.findAll(spec, PageRequest.of(0, 5));
}
```
##### 스펙 빌더 적용
```java
public class Applied {
    Specification<MemberData> spec = SpecBuilder.builder(MemberData.class)
            .ifTrue(searchRequest.isOnlyNotBlocked(),
                    () -> MemberDataSpecs.nonBlocked())
            .ifHasText(searchRequest.getName(),
                    name -> MemberDataSpecs.nameLike(searchRequest.getName()))
            .toSpec();
    List<MemberData> result = memberDataDao.findAll(spec, PageRequest.of(0, 5));
}
```
### 9. 동적 인스턴스 생성
- JPA는 쿼리 결과에서 임의의 객체를 동적으로 생성할 수 있는 기능을 제공함
- 조회 전용 모델을 만드는 이유는 표현 영역을 통해 사용자에게 데이터를 보여주기 위함
- 동적 인스턴스의 장점은 JPQL을 그대로 사용하므로 객체 기준 쿼리를 사용하면서도 지연/즉시 로딩과 같은 고민이 필요없이 데이터를 조회할 수 있다는 점
  - JPQL (Java Persistence Query Language)
    - 엔티티 객체를 조회하는 객체지향 쿼리.
    - 테이블을 대상으로 하는 것이 아닌 엔티티 객체를 대상으로 쿼리함.
    - JPA에서 제공하는 메소드 호출만으로 섬세한 쿼리 작성이 어렵다는 문제에서 탄생, 결국 SQL로 변환 됨
    - JPQL에서 엔티티의 별칭은 필수적으로 명시해야 함.
  - 즉시 로딩 , 지연 로딩
    - 즉시 로딩
      - Member를 조회 하면 연관관계에 있는 것 까지 함께 조회
    - 지연 로딩
      - Member를 조회 하면 Member만 조회하고 나머지 데이터는 조회를 미룸
     
##### JPQL
```java
public interface OrderSummaryDao extends Repository<OrderSummary, String> {
    
    @Query("""
            select new com.myshop.order.query.dto.OrderView(
                o.number, o.state, m.name, m.id, p.name
            )
            from Order o join o.orderLines ol, Member m, Product p
            where o.orderer.memberId.id = :ordererId
            and o.orderer.memberId.id = m.id
            and index(ol) - 0
            and ol.productId.id = p.id
            order by o.number.number desc
            """)
    List<OrderView> findOrderView(String ordererId);
}
```
### 10. 하이버네이트 @Subselect 사용
- 하이버네이트는 JPA 확장 기능으로 @Subselect를 제공
- @Subselect는 쿼리 결과를 @Entity로 매핑할 수 있는 기능
- @Subselect로 조회한 @Entity는 수정할 수 없다.
- @Entity 매핑 필드를 수정하면 update 쿼리가 발생하므로 이런 문제를 방지하기 위해 @Immutable을 사용??
