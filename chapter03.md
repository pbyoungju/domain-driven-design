# 3.1 Aggregate
## 상위 수준에서 모델 정리
- 상위 수준에서 모델을 정리하면 도메인 모델의 복잡한 관계를 이해하는데 도움이 된다.
<img src="https://image.pbyoungju.com/study/ddd-start/chapter-3-1.png" alt="drawing"/>


## 개별 객체 수준 모델 정리
- 개별 객체 수준에서의 모델을 바라보면 상위 수준 관계를 파악하기 어렵다.
<img src="https://image.pbyoungju.com/study/ddd-start/chapter-3-2.png" alt="drawing"/>


## Aggregate을 사용한 모델 정리
- 상위 수준, 개별 객체 수준 모두 쉽게 파악할 수 있다.
- 대부분 Aggregate 안에 있는 객체들을 함께 생성하고 함께 제거하는 등 동일한 life-cycle을 갖게 된다.
- Aggregate는 경계를 갖는다. 독립된 객체 클러스터이며 자기 자신을 관리할 뿐이다.
- 도메인 규칙, 요구사항에 따라 함께 생성되는 구성요소는 하나의 Aggregate에 속할 가능성이 높다.
- 처음에 만들 때 큰 Aggregate으로 보일 수 있지만 대부분 여러개의 Aggregate는 결국 하나의 Entity만 갖게 된다.
<img src="https://image.pbyoungju.com/study/ddd-start/chapter-3-3.png" alt="drawing"/>


# 3.2 Aggregate Root
 - Aggregate는 여러 객체로 구성되기 때문에 속한 객체들 모두 정상 상태이여야 한다.
 - 하나의 Aggregate에 속해 있는 모든 객체의 일관성을 지키기 위해 Aggregate Root를 사용한다.
 - 하나의 Aggregate에 속해 있는 모든 객체들은 Aggregate Root 객체와 연동되어 있다.

 <img src="https://image.pbyoungju.com/study/ddd-start/chapter-3-5.png" alt="drawing" width="400"/>


## 도메인 규칙과 일관성
 - Aggregate Root는 포함되어 있는 객체 클러스터와 함께 Aggregate가 제공해야할 도메인 기능을 구현하게 된다.
```java
public class Order {

    public void changeShippingInfo(ShippingInfo newShippingInfo) {
        verifyNotYetShipped();
        setShippingInfo(newShippingInfo);
    }

    private void verifyNotYetShipped() {
        if (state != OrderState.PAYMENT_WAITING && state != OrderState.PREPARING) {
            throw new IllegalStateException("already shipped");
        }
    }
}
```

 - Aggregate 외부에서 Aggregate에 속한 객체를 직접 변경하면 안된다.
```java
// Aggregate가 제공하는 도메인 기능을 이용하지 않고 직접 set으로 변경하면 논리적인 데이터 일관성이 깨질 수 있다.
ShippingInfo si = order.getShippingInfo();
si.setAddress(newAddress);
```

- 도메인 일관성을 지키는 로직을 Application에 구현하게 된다면 중복코드가 많아질 수 있다.
```java
ShippingInfo si = order.getShippingInfo();

if (state != OrderState.PAYMENY_WAITING && state != OrderState.PREPARING) {
    throw new IllegalArgumentException();
}

si.setAddress(newAddress);
```

### 불필요한 중복을 막고 Aggregate Root를 통해서만 도메인 로직 구현
 - 단순히 필드를 변경하는 set을 public 하지 않게 한다.
 - Value Object는 불변으로 한다.
 - set형식의 이름을 갖는 메서드가 아닌 cancel, changePassword와 같은 이름을 사용한다.

```java
// Value Object를 변경하지 못하고 새로 생성하는 것으로 강제할 수 있다.
public class Order {

    private ShippingInfo shippingInfo;

    public void changeShippingInfo(ShippingInfo newShippingInfo) {
        verifyNotYetShipped();
        setShippingInfo(newShippingInfo);
    }

    private void setShippingInfo(ShippingInfo newShippingInfo) {
        this.shippingInfo = newShippingInfo
    }
}
```

## 3.2.2 Aggregate Root의 기능 구현
- Aggregate Root는 Aggregate 내부의 다른 객체를 조합해서 기능을 완성한다.
```java
public class Order {
    private Money totalAmounts;
    private List<OrderLine> orderLines;

    private void calculateTotalAmounts() {
        int sum = orderLines.stream()
                    .mapToInt(o1 -> o1.getPrice() * o1.getQuantity())
                    .sum();

        this.totalAmounts = new Money(sum);
    }
}
```
```java
public class Member {
    private Password password;

    public void changePassword(String currentPassword, String newPassword) {
        if (!password.match(currentPassword)) {
            throw new PasswordNotMatchException();
        }

        this.password = new Password(newPassword);
    }
}
```
- Aggregate Root는 기능 실행을 위임하기도 한다.
```java
public class OrderLines {
    private List<OrderLine> lines;

    public Money getTotalAmounts() {
        // ... 구현
    }

    public void changeOrderLines(List<OrderLine> newLines) {
        this.lines = newLines;
    }
}
```
```java
public class Order {
    private Money totalAmount;
    private OrderLines orderLines;

    public void changeOrderLines(List<OrderLine> newLines) {
        this.orderLines.changeOrderLines(newLines);
        this.totalAmount = this.orderLines.getTotalAmount();
    }
}
```
- 
```java
OrderLines lines = order.getOrderLines();

// 외부에서 Aggregate 내부 상태 변경
// Order의 totalAmounts의 값과 OrderLines이 일치하지 않게 됨
// .changeOrderLines()를 protected로 한정하는 방법도 있다.
lines.changeOrderLines(newOrderLines);
```

## 3.2.3 트랜잭션 범위
 - 트렌잭션은 작을수록 좋다.
 - 트랜잭션과 Aggregate은 1:1이 좋다.

### Aggregate 내부에서 다른 Aggregate를 수정하면 안된다.
```java
public class Order {
    private Orderer orderer;

    public void shipTo(ShippingInfo newShippingInfo, boolean useNewShippingAddrAsMemberAddr) {
        verifyNotYetSupport();
        setShippingInfo(newShippingInfo);
        if (useNewShippingAddrAsMemberAddr) {
            // member은 다른 Aggregate
            // 다른 Aggregate 상태를 바꾸면 안됨
            // Aggregate 간 결합도가 높아져서 나중에 유지보수가 힘들어진다.
            orderer.getMember().changeAddress(newShippingInfo.getAddress());
        }
    }
}
```

### Application layer에서 2개 이상의 Aggregate을 수정해야 한다.
```java
public class ChangeOrderService {

    @Transaction
    public void changeShippingInfo(OrderId id, ShippingInfo newShippingInfo, booealn useNewShippingAddrAsMemberAddr) {
        Order order = orderRepository.findById(id);
        if (order == null) throw new OrderNotFoundException();
        order.shipTo(newShippingInfo);
        if (useNewShippingAddrAsMemberAddr) {
            Member member = findMember(order.getOrderer());
            member.changeAddress(newShippingInfo.getAddress());
        }
    }
}
```

# 3.3 Repository와 Aggregate
 - Order와 OrderLine을 테이블 단위로 분리해도 Repository를 따로 만들지 않을 수 있다.
 - Order은 Aggregate Root고 OrderLine은 속하는 구성요소이다.
 - Aggregate Root를 저장할 때 속해 있는 모든 구성요소도 같이 영속화 되어야 한다.
```java
// repository는 완전한 order을 제공해야 한다.
Order order = orderRepository.findById(orderId);

// order가 완전한 Aggregate가 아니면
// 기능 실행 도중 NullPointerException과 같은 문제가 발생한다.
order.cancel();
```

# 3.4 ID를 이용한 Aggregate 참조
 - Aggregate 안의 구성요소는 다른 Aggregate Root를 참조할 수 있다.
```java
public class Order {
    private Orderer orderer;
}

public class Orderer {
    private Member member;
    private String name;
}
```

```java
public class Member {

}
```
 - 필드를 이용해서 다른 Aggregate를 편리하게 직접 참조할 수 있지만 안좋은 점도 있다.
```java
// 편한 탐색 오용
// 성능에 대한 고민
// 확장 어려움
order.getOrderer().getMember().getId();
```
- 하나의 Aggregate가 관리하는 범위는 자기 자신으로 한정해야지 좋다.
```java
public class Order {
    private Orderer orderer;

    public void changeShippingInfo(ShippingInfo newShippingInfo, boolean useNewShippingAddrAsMemberAddr) {
        if (useNewShippingAddrAsMemberAddr) {
            // 1개의 Aggregate 내부에서 다른 Aggregate에 접근할 수 있으면
            // 구현이 쉬워져서 다른 Aggregate 상태를 변경하는 유혹에 빠지기 쉽다.
            // JPA Lazy, Eager 로딩 전략을 잘 세워야 한다.
            // 도메인 별로 다른 DB를 가져가게 될 수도 있다.
            orderer.getMember().changeAddress(newShippingInfo.getAddress());
        }
    }
}
```

 - 직접참조가 아닌 ID를 이용해서 Aggregate 참조
```java
public class Order {
    private Orderer orderer;
}

public class Orderer {
    private MemberId memberId;
    private String name;
}
```
```java
public class Member {

    // 장점은 Aggregate간 물리적인 연결을 대신하기 때문에 결합도를 낮춰준다.
    private MemeberId id;
}
```
- ID 외의 나머지 필드들은 repository를 이용하여 load하면 된다.
```java
public class ChangeOrderService {

    @Transactional
    public void changeShippingInfo(OrderId id, ShippingInfo newShippingInfo, boolean useNewShippingAddrAsMemberAddr) {
        Order order = orderRepository.findById(id);
        if (order == null) throw new OrderNotFoundException();
        
        order.changeShippingInfo(newShippingInfo);
        if (useNewShippingAddrAsMemberAddr) {
            // 자연스럽게 지연 로딩을 하게 된다.
            Member member = memberRepository.findById(
                order.getOrderer().getMemberId());
        
            member.changeAddress(newShippingInfo.getAddress());
        }
    }
}
```

## 3.4.1 ID를 이용한 참조와 조회 성능
 - N개의 데이터를 참조하는 하나의 Aggregate를 Read할 때 성능이슈가 발생할 수 있다.
 - Lazy 로딩 + JPQL fetch join으로 이슈를 해결할 수 있다. (ID 참조가 아닌 직접 참조가 되어버린다.)
```java
Member member = memberRepository.findById(ordererId);
List<Order> orders = orderRepository.findByOrderer(ordererId);
List<OrderView> dtos = orders.stream()
        .map(order -> {
            ProductId prodId = order.getOrderLines().get(0).getProductId();
            Product product = productRepository.findById(prodId);
            return new OrderView(order, member, product);
        })
        .collect(toList());
```

### 조회 전용 쿼리를 만든다.
- 조회를 위한 별도 DAO를 만들고 SQL JOIN을 이용한다.
```java
@Repository
public class JpaOrderViewDao implements OrderViewDao {

    @PesistenceContext
    private EntityManager em;

    @Override
    public List<OrderView> selectByOrderer(String ordererId) {
        String selectQuery =
            "select new com.myshop.order.application.dto.OrderView(o, m, p) " +
            "from Order o join o.orderLines o1, Member m, Product p " +
            "where o.orderer.memberId.id = :ordererId " +
            "and o.orderer.memberId = m.id " +
            "and index(o1) = o " +
            "and o1.productId = p.id " +
            "order by o.number.number desc";

        TypedQuery<OrderView> query =
            em.createQuery(selectQuery, OrderView.class);
        
        query.setParameter("ordererId", ordererId);

        return query.getResultList();
    }
}
```

# 3.5 Aggregate 간 집합 연관
 - 카테고리 : 상품 -> 1 : N
```java
public class Category {
    
    private Set<Product> products;
}
```


<img src="https://image.pbyoungju.com/study/ddd-start/chapter-3-4.png" alt="drawing"/>

 - 실제 요구사항과 다를 수 있다. 보통은 페이징처리로 끊어서 보여주게 된다.
```java
public class Category {

    // 실제 요구사항은 보통 n개를 모두 가져오지 않는다.
    private Set<Product> products;

    // 아래 처럼 구현해도 DB에서 전부 다 가져오기 때문에 성능에 문제가 생긴다.
    public List<Product> getProducts(int page, int size) {
        List<Product> sortedProducts = sortById(products);
        return sortedProducts.subList((page - 1) * size, page * size);
    }
}
```
 - 1:N의 관계를 N:1로 변경한다.
```java
public class Product {

    private CategoryId categoryId;
}
```
```java
public class ProductListService {
    public Page<Product> getProductOfCategory(Long categoryId, int page, int size) {
        Category category = categoryRepository.findById(categoryId);
        checkCategory(category);
        
        List<Product> products =
            productRepository.findByCategoryId(category.getId(), page, size);
        int totalCount = productRepository.countsByCategoryId(category.getId());
        return new Page(page, size, totalCount, products);
    }
}
```
 - 상품과 카테고리는 M:N인 경우도 있다.
```java
public class Product {

    // 상품 목록 리스트에서 하나의 상품에 모든 카테고리를 보여주지는 않는다.
    // 상품 상세화면에서나 모든 카테고리를 보여준다.
    // 지연로딍 하면 된다.   
    private Set<CategoryId> categoryIds;
}
```
- M:N 테이블 매핑 예제
[이미지]
```java
@Entity
@Table(name = "product")
public class Product {

    @EmbeddedId
    private ProductId id;

    @ElementCollection
    @CollectionTable(
        name = "product_category",
        joinColumns = @JoinColumn(name = "product_id")
    )
    private Set<CategoryId> categoryIds;
}
```

```java
@Repository
public class JpaProudctRepository implements ProductRepository {

    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public List<Product> findByCategoryId(CategoryId catId, int page, int size) {
        TypedQuery<Product> query = entityManager.createQuery(
            "select p from Product p " +
            "where :catId member of p.categoryIds order by p.id.id desc",
            Product.class);

        query.setParameter("catId", catId);
        query.setFirstResult((page -1) * size);
        query.setMaxResults(size);
        
        return query.getResultList();
    }
}
```
 - JPQL member of를 이용하여 Category(1)에 속한 Product(N) 목록을 구함
 - `:catId member of p.categoryIds`는 categoryIds 컬렉션에 catId가 존재하는지 검사하는 구문

> 복잡하게 생각하지 말자. 버그만 생길 뿐이다. 그냥 read는 전용모델 만들어서 한방쿼리 날리는게 좋다.

# 3.6 Aggregate를 factory로 사용하기
 - 상품 등록 기능에는 차단된 상점일 경우 막아야 한다.
```java
public class RegisterProductService {

    // product를 생성할 때 처리되는 도메인 로직이 응용 서비스단에 노출되어 있어서 안좋은 코드이다. 
    public ProductId registerNewProduct(NewProductRequest req) {
        Store store = storeRepository.findById(req.getStoreId());
        checkNull(store);

        if (account.isBlocked()) {
            throw new StoreBlockedException();
        }

        ProductId id = productRepository.nextId();
        Product product = new Product(id, store.getId(), ...);
        productRepository.save(product);
        return id;
    }
}
```

- Store Aggregate의 `createProduct()`는 Product Aggregate를 생성하는 팩토리 역할을 한다.
```java
public class Store {
    public Product createProduct(ProductId newProductId, ...) {
        if (this.isBlocked()) throw new StoreBlockedException();
        return new Product(newProductId, getId(), ...);
    }
}
```

- 아래처럼 코드를 변경했을 때 이점은 Product 생성 여부 확인 로직을 도메인에서만 체크하기 때문에 변경할 때 수정되는 지점을 줄일 수 있다.
```java
public class RegisterProductService {

    public ProductId registerNewProduct(NewProductRequest req) {
        Store store = storeRepository.findById(req.getStoreId());
        checkNull(store);

        ProductId id = productRepository.nextId();
        Product product = store.createProduct(id, ...);
        productRepository.save(product);
        return id;
    }
}
```

- Store Aggregate가 Product Aggregate를 생성할 때 많은 정보를 알아야 한다면 별도의 팩토리 클래스를 이용한다.
```java
public class Store {
    public Product createProduct(ProductId newProductId, ProductInfo pi) {
        if (isBlocked()) throw new StoreBlockedException();
        return ProductFactory.create(newProductId, getId(), pi);
    }
}
```