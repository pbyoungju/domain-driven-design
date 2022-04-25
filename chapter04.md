# 4.1 JPA를 이용한 리포지터리 구현
객체 기반의 도메인 모델과 관계형 데이터 모델 간의 매핑을 처리하기 위해 JPA를 사용해본다.

## 4.1.1 모듈 위치
### 위치
- repository interface -> agreegate와 같이 도메인 영역 위치
- repository implementation -> 인프라스트력쳐 영역에 위치
> repository implementation를 domain.impl과 같은 패키지에 위치해도 된다.

## 4.1.2 repository 기본 기능 구현
### 기본 명세
- ID로 agreegate 조회
- agreegate 저장
```java
public interface OrderRepository {
    Order findById(OrderNo no);
    void save(Order order);
}
```

<img src="https://image.pbyoungju.com/study/ddd-start/chapter-4-1-2.png" alt="drawing" width="400"/>

<br />

### JPA를 이용한 구현체
```java
public class OrderRepositoryImpl implements OrderRepository {
    
    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public Order findById(OrderNo id) {
        return entityManager.find(Order.class, id);
    }

    @Override
    public void save(Order order) {
        entityManager.persist(order);
    }
}
```

### Spring Transaction을 이용한 수정된 aggregate 영속화
- update를 하지 않아도 변경된다.
```java
public class ChangeOrderService {
    
    @Transactional
    public void changeShippingInfo(OrderNo no, ShippingInfo newShippingInfo) {
        Optional<Order> orderOpt = orderRepository.findById(no);
        Order order = orderOpt.orElseThrow(() -> new OrderNotFoundException());
        order.changeShippingInfo(newShippingInfo);
    }
}
```

- id가 아닌 조건으로 조회할 경우 및 JPQL 이용
```java
public interface OrderRepository {
    
    List<Order> findByOrderedId(String orderedId, int startRow, int size);
}
```

```java
public class OrderRepositoryImpl implements OrderRepository {

    @Override
    public List<Order> findByOrderedId(String orderedId, int startRow, int fetchSize) {
        TypedQuery<Order> query = entityManager.createQuery(
            "select o from Order o " +
                "where o.orderer.memberId.id = :ordererId " +
                "order by o.number.number desc",
                Order.class);
        query.setParameter("orderedId", orderedId);
        query.setFirstResult(startRow);
        query.setMaxResults(fetchSize);
        return query.getResultList();
    }
}
```

- Aggregate를 삭제하는 기능(soft delete 하는 경우가 많다.)
```java
public interface OrderRepository {

    public void delete(Order order);
}
```

```java
public class OrderRepository implements OrderRepository {

    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public void delete(Order order) {
        this.entityManager.remove(order)
    }
}
```

<br />

# 4.2 SPRING DATA JPA를 이용한 리포지터리 구현
- SPRING DATA JPA는 규칙에 맞게 repository 인터페이스를 정의하면 빈으로 등록 후 해당 인터페이스 구현을 자동으로 해준다.

## 예시코드
### Entity
```java
@Entity
@Table(name = "purchase_order")
@Access(AccessType.FIELD)
public class Order {

    @EmbeddedId
    private OrderNo number;
}

```

### OrderRepository
```java
public interface OrderRepository extends Repository<Order, OrderNo> {
    
    Optional<Order> findById(OrderNo id);

    void save(Order order);
}
```

### OrderService
```java
@Service
public class CancelOrderService {
    
    private final OrderRepository orderRepository

    public CancelOrderSErvice(OrderRepository orderRepository) {
        this.orderRepository = orderRepository
    }

    @Transational
    public void cancel(OrderNo orderNo, Canceller canceller) {
        Order order = orderRepository.findById(orderNo)
                        .orElseThrow(() -> new NoOrderException());
        
        if (!cancelPolicy.hasCancellationPermission(order, canceller)) {
            throw new NoCancellablePermission();
        }

        order.cancle();
    }
}
```

## entity 저장
 - Order save(Order entity)
 - void save(Order entity)

## entity 조회(식별자 이용)
 - Order findById(OrderNo id)
 - Optional<Order> findById(OrderNo id)

## entity 조회(식별자 아닌 다른 속성)
 - findByOrderer(Orderer orderer)

## entity 조회(객체 안의 객체)
 - List<Order> findByOrdererMemberId(MemberId memberId)

## entity 삭제
 - void delete(Ordered order)
 - void deleteById(OrderNo id)

<br />

# 4.3 매핑구현
## Aggregate와 table 관계
 - aggregate root는 entity이므로 @Entity로 매핑
 - value object는 @Embeddable 매핑
 - value object properties는 @Embedded로 매핑

<img src="https://image.pbyoungju.com/study/ddd-start/chapter-4-3-1.png" alt="drawing" width="400"/>

### ROOT ENTITY -> @Entity
```java
import javax.persistence.Entity;

@Entity
@Table(name = "purchase_order")
public class Order {

    @Embedded
    private Ordered orderer;

    @Embedded
    private ShippingInfo shippingInfo;
}
```

### VO -> @Embeddable
 - Orderer -> MemberId
```java
import javax.persistence.Entity;

@Embeddable
public class Orderer {

    @Embedded
    @AttributeOverides(
        @AttributeOverride(name = "id", column = @Column(name = "orderer_id"))
    )
    private MemberId memberId;

    @Column(name = "orderer_name")
    private String name;
}
```
```java
@Embeddable
public class MemberId implements Serializable {

    @Column(name = "member_id")
    private String id;
}
```

 - ShippingInfo -> Ordered
```java
@Embeddable
public class MemberId implements Serializable {

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "zipCode", column = @Column(name = "shipping_zipcode")),
        @AttributeOverride(name = "address1", column = @Column(name = "shipping_addr1")),
        @AttributeOverride(name = "address2", column = @Column(name = "shipping_addr2"))
    })
    private Address address;

    @Column(name = "shipping_message")
    private String message;

    @Embedded
    private Receiver receiver;
}
```

## 4.3.2 기본 생성자
 - entity, vo는 constructor로만 값을 할당받아 immutable하게 한다.
 - 하지만 JPA는 객체들을 프록시로 미리 만들기 때문에 no-argument constructor가 필요하다.
```java
@Embeddable
public class Receiver {

    @Column(name = "receiver_name")
    private String name;

    @Column(name = "receiver_phone")
    private String phone;

    // package-private
    protected Receiver() {}

    public Receiver(String name, String phone) {
        this.name = name;
        this.phone = phone;
    }
}
```

## 4.3.3 필드 접근 방식 사용
- 예제에 사용될 ENUM
```java
public enum OrderState {
    "DELIVERY", "FINISH", "..."
}
```

- 메서드 방식
```java
@Entity
@Access(AccessType.PROPERTY)
public class Order {

    private OrderState state;

    @Column(name = "state")
    @Enumerate(EnumType.STRING)
    public OrderState getState() {
        return state;
    }
}
```

- 필드 방식
```java
@Entity
@Access(AccessType.FIELD)
public class Order {

    @EmbeddedId
    private OrderNo number;

    @Column(name = "state")
    @Enumerated(EnumType.STRING)
    private OrderState state;
}
```

## 4.3.4 AttributeConverter를 이용한 Value Mapping 처리
- 2개 이상의 프로퍼티를 가진 Value 타입을 조합하여 1개의 컬럼에 Mapping할 때 사용
```java
public class Length {
    private int value;
    private String unit;
}
```
```sql
-- 1000 mm
WIDTH VARCHAR(200)
```
### AttributeConverter
- 아래 코드와 같이 X는 Value Object 타입, Y는 Column 타입
- `convertToDatabaseColumn()`은 VO -> Column
- `convertToEntityAttribute()`는 Column -> VO
```java
package javax.persistence;

public interface AttributeConverter<X, Y> {
    
    public Y convertToDatabaseColumn(X attribute);

    public X convertToEntityAttribute(Y dbData);
}
```

### VO Money 관련 코드
```java
package com.myshop.common.jpa;

import com.myshop.common.model.Money;

import javax.persistence.AttributeConverter;
import javax.persistence.Converter;

@Converter(autoApply = true)
public class MoneyConverter implements AttributeConverter<Money, Integer> {

    @Override
    public Integer convertToDatabaseColumn(Money money) {
        return money == null ? null : money.getValue();
    }

    @Override
    public Money convertToEntityAttribute(Integer value) {
        return value == null ? null : new Money(value);
    }
}
```

- @Converter 애노테이션의 autoApply가 true면 model에 연관되어 있는 모든 money는 해당 Conveter을 이용

```java
@Entity
@Table(name = "purchase_order")
public class Order {

    @Column(name = "total_amounts")
    private Money totalAmounts;
}
```

- autoApply가 false면 해당 converter을 사용하기 위해 클래스의 필드에 지정해줘야 한다.

```java
import javax.persistence.Convert;

public class order {

    @Column(name = "total_amount")
    @Convert(converter = MoneyConverter.class)
    private Money totalAmounts;
}
```

## 4.3.5 Value Collection, 별도 Table Mapiing
- Order Entity와 OrderLine VO는 1 : N 관계

```java
@Entity
@Table(name = "purchase_order")
public class Order {

    @EmbeddedId
    private OrderNo number;

    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(
        name = "order_line", 
        joinColumns = @JoinColumn(name = "order_number"))
    @OrderColumn(name = "line_idx")
    private List<OrderLine> orderlines;
}
```
```java
@Embeddable
public class OrderLine {

    @Embedded
    private ProductId productId;

    @Column(name = "price")
    private Money price;

    @Column(name = "quantity")
    private int quantity;

    @Colmn(name = "amounts")
    private Money amounts;
}
```

## 4.3.6 Value Collection, 1개 Column Mapping
 - 도메인 모델에 N개의 이메일 주소를 Set으로 보관, DB에는 하나의 컬럼에 Comma로 구분하고 저장
 - Value Collection을 표현하는 새로운 Value Type을 만들어 AttributeConverter을 이용

```java
public class EmailSet {

    private Set<Email> emails = new HashSet<>();

    public EmailSet(Set<Email> emails) {
        this.emails.addAll(emails);
    }

    public Set<Email> getEmails() {
        return Collections.unmodifiableSet(emails);
    }
}
```

```java
public class EmailSetConverter implements AttributeConverter<EmailSet, String> {

    @Override
    public String convertToDatabaseColumn(EmailSet attribute) {
        if (attribute == null) return null;

        return attribute.getEmails().stream()
                    .map(email -> email.getAddress())
                    .collect(Collectors.joining(","))
    }

    @Override
    public EmailSet convertToEntityAttribute(String dbData) {
        if (dbData == null) return null;
        String[] emails = dbData.split(",");
        Set<Email> emailSet = Arrays.stream(emails)
                .map(value -> new Email(value))
                .collect(toSet());

        return new EmailSet(emailSet);
    }
}
```

```java
@Entity
@Table(name = "purchase_order")
public class Order {

    @Column(name = "emails")
    @Converter(converter = EmailSetConverter.class)
    private EmailSet emailSet;
}
```

## 4.3.7 Value Object를 이용한 ID Mapping
- 지금까지 봤던 OrderNo, MemberId 등이 모두 VO 타입이다.
- @Id 대신 @EmbeddedId를 사용한다.
```java
@Entity
@Table(name = "purchase_order")
public class Order {

    @EmbeddedId
    private OrderNo number;
}

@Embeddable
public class OrderNo implements Serializable {

    @Column(name = "order_number")
    private String number;
}
```

- 레거시 주문번호와 신 주문번호를 구분할 때 유용할 수 있다.
```java
@Embeddable
public class OrderNo implements Serializable {
    
    @Column(name = "order_number")
    private String number;

    public boolean isNewOrderNo() {
        return this.number.startsWith("N");
    }
}
```
```java
if (order.getNumber().isNewOrderNo()) {
    ...
}
```

## 4.3.8 별도 테이블에 저장하는 Value Mappings
- Aggregate에서 Root Entity를 제외한 나머지는 대부분 VO
- 주문 Aggregate도 OrderLine을 별도 테이블에 저장하지만 OrderLine 자체는 엔티티가 아님
- Article - ArticleContent도 1:1 매핑이 아닌 Root Entity와 VO의 관계이다.

<img src="https://image.pbyoungju.com/study/ddd-start/chapter-4-3-8.png" alt="drawing" width="400"/>

### SecondaryTable을 이용한 VO 매핑
```java
@Entity
@Table(name = "article")
@SecondaryTable(
    name = "article_content",
    pkJoinColumns = @PrimaryKeyJoinColumn(name = "id")
)
public class Article {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;

    @AttributeOverrides({
        @AttributeOverride(
            name = "content",
            column = @Column(table = "article_content", name = "content")),
        @AttributeOverride(
            name = "contentType",
            column = @Column(table = "article_content", name = "content_type"))
    })
    @Embedded
    private ArticleContent content;
}
```
게시글 리스트를 보여주는 화면에서는 ArticleContent의 데이터는 필요없다. 하지만 `@SecondaryTable`을 사용하면 
eager loading 전략으로 join문이 나가서 불필요한 데이터를 얻어온다. 이를 Entity로 만들어 OneToOne매핑을 사용하여 Lazy Loading을 할 수 있지만 
좋지 않은 방법이다.

## 4.4 Aggregate 로딩 전략
 - Aggregate를 로딩하면 Root에 연관된 하위 객체가 모두 로딩되어야 한다.

### FetchType.Eager
- 조회 시점에서 FetchType.EAGER을 사용하면 하위 객체들이 모두 로딩된다.
```java
@OneToMany(
    cascade = {CascadeType.PERSIST, CascadeType.REMOVE},
    orphanRemoval = true, 
    fetch = FetchType.EAGER
)
@JoinColumn(name = "product_id")
@OrderColumn(name = "list_idx")
private List<Image> images = new ArrayList<>();

@ElementCollection(fetch = FetchType.EAGER)
@CollectionTable(
    name = "order_line", 
    joinColumns = @JoinColumn(name = "order_number")
)
@OrderColumn(name = "line_idx")
private List<OrderLine> orderLines;
```

### 카타시안 곱의 문제
```java
@Entity
@Table(name = "product")
public class Product {

    @OneToMany(
        cascade = { CascadeType.PERSIST, CascadeType.REMOVE },
        orphanRemoval = true,
        fetch = FetchType.EAGER
    )
    @JoinColumn(name = "product_id")
    @OrderColumn(name = "list_idx")
    private List<Image> images = new ArrayList<>();

    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(
        name = "product_option",
        joinColumns = @JoinColumn(name = "product_id")
    )
    @OrderColumn(name = "list_idx")
    private List<Option> options = new ArrayList<>();
}
```

### Transation을 사용하여 LAZY LOADING
```java
@Transational
public void removeOptions(ProductId id, int optIdxToBeDeleted) {
    Product product = productRepository.findById(id);
    product.removeOption(optIdxToBeDeleted);
}
```

```java
@Entity
public class Product {

    @ElementCollection(fetch = FetchType.LAZY)
    @CollectionTable(
        name = "product_option", 
        joinColumns = @JoinColumns(name = "product_id"))
    @OrderColumn(name = "list_idx")
    private List<Option> options = new ArrayList<>();

    public void removeOption(int optIdx) {
        this.options.remove(optIdx);
    }
}
```

즉시, 지연 로딩은 @Entity, @Embeddable에 따라 다르게 동작하고 JPA Provider에 따라 또 다르다.

## 4.5 Aggregate 영속성 전파
### 애그리거트가 완전한 상태여야 한다는 것은 저장하고 삭제할 때도 하나로 처리해야함을 의미
- 저장 메서드는 AggregateRoot만 저장하면 안되고 AggregateRoot에 속한 모든 객체를 저장
- 삭제 메서드는 AggregateRoot에 속한 모든 객체를 삭제

### @Embeddable 매핑
 - @Embeddable은 함께 저장되고 삭제되므로 Cascade 옵션이 필요 없다.

### @Entity 매핑
 - @Entity는 cascade 속성을 추가로 설정해야 한다.
```java
@OneToMany(cascade = { CascadeType.PERSIT, CascadeType.REMOVE }, orphanRemoval = true)
@JoinColumn(name = "product_id")
@OrderColumn(name = "list_idx")
private List<Image> images = new ArrayList<>();
```
<br />


## 4.6 식별자 생성 기능
 - 응용 서비스에서 생성
```java
public class CreateProductService {
    @Autowired private ProductIdService idService;
    @Autowired private ProductRepository productRepository;

    @Transational
    public ProductId createProduct(ProductCreationCommand cmd) {
        ProductId id = productIdService.nextId();
        Product product = new Product(id, cmd.getDetail(), cmd.getPrice());
        productRepository.save(product);
        return id;
    }
}
```
```java
public class OrderIdService {
    public OrderId createId(UserId userId) {
        if (userId == null) 
            throw new IllegalArgumentException("invalid userid: " + userId);
        
        return new OrderId(userId.toString() + "-" + timestamp());
    }

    private String timestamp() {
        return Long.toString(System.currentTimeMillis());
    }
}
```
```java
public interface ProductRepository {
    ProductId nextId();
}
```
 - 도메인 로직으로 생성
```java
public class ProductIdService {

    public ProductId nextId() {
        return new ProductId().nextId();
    }
}
```
 - DB를 이용한 일련번호 생성
```java
@Entity
@Table(name = "article")
public class Article {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    public Long getId() {
        return id;
    }
}
``` 
- DB 자동증가 컬럼을 사용할 때는 .save() 후 식별자를 구할 수 있다.
```java
public class WriteArticleService {
    private ArticleRepository articleRepository;

    public Long write(NewArticleRequest req) {
        Article article = new Article("title", new ArticleContent("content", "type"));
        articleRepository.save(article);
        return article.getId();
    }
}
```

## 4.7 도메인 구현과 DIP
- Entity와 Repository 같은 경우 순수 POJO가 아니고 JPA 기술에 의존적인 Annotation으로 구성되어 있다.
- 해당 객체들은 JPA에 의존적이므로 infrastructrue 영역에 포함되어야 한다.

<img src="https://image.pbyoungju.com/study/ddd-start/chapter-4-7-1.png" alt="drawing" width="400"/>

- DIP를 적용하는 이유는 저수준 구현이 변경되어도 고수준이 영황 받지 않도록 하기 위함이다.
- Repository, domain은 거의 바뀌지 않는다. 그래서 어느정도 타협안으로 위의 예시처럼 구현하는게 제일 좋은 방법 같다.