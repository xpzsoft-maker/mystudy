## 一、问题背景
&emsp;&emsp;spring-boot-starter-data-jpa的数据持久化工具默认为Hibernate，Hibernate在加载实体对象时，
如果实体中有外键关联（`OneToOne`、`OneToMany`、`ManyToOne`、`ManyToMany`），外键对象默认采用懒加载（`FetchType.LAZY`），
懒加载设计的目的是为了按需加载，避免不必要的sql查询，从而提供程序性能。但是在设计SpringBoot的RESTful API时，
默认采用jackson作为实体的序列化与反序列化工具，而jackson在序列化包含懒加载外键实体时，会激活懒加载对象，
查询出多余的数据，破坏了懒加载的按需加载特性。    
&emsp;&emsp;为了更好地描述这个问题，看以下示例：购物车ShoppingCart与商品Good为OneToMany关系，商品具有
明细GoodDetails，Good与GoodDetails为OneToOne关系：
```java
@Data
@Entity
@Table(name = "SHOPPING_CART", schema = "testDB")
public final class ShoppingCart {
  /**
  * 主键
  */
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private long id;


  /**
  * 购物车中的商品
  */
  @OneToMany(mappedBy = "shoppingCart", cascade = CascadeType.ALL)
  private Set<Good> goods;

}
```
```java
@Data
@Entity
@Table(name = "GOOD", schema = "testDB")
public final class Good {
  /**
  * 主键
  */
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private long id;

  /**
  * 购物车中商品名称
  */
  private String name;

  /**
  * 购物车中商品数量
  */
  private long count;

  /**
  * 购物车中商品单价
  */
  private double price;


  /**
  * 购物车
  */
  @ManyToOne
  @JoinColumn(referencedColumnName = "id")
  private ShoppingCart shoppingCart;

  /**
  * 商品明细
  */
  @OneToOne(mappedBy = "good", cascade = CascadeType.ALL)
  private GoodDetails goodDetails;

}
```

```java
@Data
@Entity
@Table(name = "GOOD_DETAILS", schema = "testDB")
public final class GoodDetails {
  /**
  * 主键
  */
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private long id;

  /**
  * 商品描述
  */
  private String desc;

  /**
  * 商品图像地址
  */
  private String imgPath;


  @OneToOne
  @JoinColumn(referencedColumnName = "id")
  private Good good;

}
```
&emsp;&emsp;我们设计一个API接口，希望查询当前购物车及其中的商品列表，不需明细商品明细：
```java
@RestController
@RequestMapping("/shoppingCart")
public final class ShoppingCartController {
  private ShoppingCartService shCaService;

  @Autowired
  private ShoppingCartController(ShoppingCartService shCaService) {
    this.shCaService = shCaService;
  }

  @GET("/{cartId}")
  public ShoppingCart getShoppingCart(@PathVariable("cartId") cartId) {
    ShoppingCart cart = this.shCaService.getShoppingCart(cartId);
    // 断点tag： 当运行到此处时，程序运行正常
    return cart;
  }
}
```
```java
public interface ShoppingCartService {
  ShoppingCart getShoppingCart(long cartId);
}
```
```java
@Service
public final class ShoppingCartServiceImpl 
    implements ShoppingCartService {
  private ShoppingCartRepository shCaRepository;

  @Autowired
  private ShoppingCartServiceImpl(ShoppingCartRepository shCaRepository) {
    this.shCaRepository = shCaRepository;
  }

  public ShoppingCart getShoppingCart(long cartId) {
    ShoppingCart cart = this.shCaRepository.findById(cartId);
    
    Optional.ofNullable(cart).orElseThrow(
      () -> new NotFoundException(
        String.format("购物车id=%d不存在", cartId)));
    
    // 激活懒加载，获取商品列表
    cart.getGoods().size();

    return cart;
  }
}
```
```java
public interface ShoppingCartRepository extends
        JpaRepository<ShoppingCart, Long> {
  ShoppingCart findById(cartId);
}
```
&emsp;&emsp;此时，一切看起来都不错，但当我们调用`/shoppingCart/{cartId}`时，直到在ShoppingCartController中标注为“断点:tag”处，
程序任然正常运行，但是return cart后，jackson序列化cart报错：**`实体间循环引用，无限递归将导致堆栈溢出`**，
此问题可以通过以下两个方法解决：
1. 对外键关联的父对象设置为@Jsonignore（不推荐）
```java
@Data
@Entity
@Table(name = "GOOD", schema = "testDB")
public final class Good {
  //其它未改动处省略
  ...

  /**
  * 购物车
  */
  @Jsonignore
  @ManyToOne
  @JoinColumn(referencedColumnName = "id")
  private ShoppingCart shoppingCart;

  //其它未改动处省略
  ...
}
```
2. 设置@JsonIdentityInfo（<b>推荐</b>）
```java
@JsonIdentityInfo(generator = ObjectIdGenerators.PropertyGenerator.class, property = "id")
public class BaseEntity{
  private long id;

  public long getId() {
    return id;
  }
}
```
```java
@Data
@Entity
@Table(name = "SHOPPING_CART", schema = "testDB")
public final class ShoppingCart extends BaseEntity {
  @Override
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  public long getId() {
    return super.getId();
  }

  //其它未改动处省略
  ...
}

@Data
@Entity
@Table(name = "GOOD", schema = "testDB")
public final class Good extends BaseEntity{
  @Override
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  public long getId() {
    return super.getId();
  }

  //其它未改动处省略
  ...
}

@Data
@Entity
@Table(name = "GOOD_DETAILS", schema = "testDB")
public final class GoodDetails extends BaseEntity{
  @Override
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  public long getId() {
    return super.getId();
  }

  //其它未改动处省略
  ...
}
```

&emsp;&emsp;修改后再调用`/shoppingCart/{cartId}`，可能会出现以下两中情况：
1. 报错：**`jackson尝试序列化Good实体中的goodDetails对象，但是该对象目前是一个Hibernate懒加载代理，序列化失败`**
2. 执行成功，jackson成功激活了Hibernate懒加载代理，将我们不需要的goodDetails也查询出来，返回如下json字符串：
```json
{
  "id": 123,
  "goods": [
    {
      "id": 11,
      "name": "华为手机P30",
      "price": 3000.00,
      "count": 10,
      "goodDetails": { //增加了多余的sql查询，返回了我们不需要的数据
        "id": 111,
        "desc": "这是一个像素很高的手机"
      }
    },
    {
      "id": 12,
      "name": "京东跑步鸡",
      "price": 128,
      "count": 10,
      "goodDetails": { //增加了多余的sql查询，返回了我们不需要的数据
        "id":121,
        "desc": "跑步鸡是只好鸡，用茶油做味道好极了，就是有点儿贵"
      }
    }
  ]
}
```
&emsp;&emsp;很明显，不管是以上哪种情况，都不符合我们的期望。接下来，我们将给出解决方案。

## 二、解决方案
&emsp;&emsp;接下来我们将给出“按需加载”与“按非需加载”两种解决方案。
### 2.1 按需加载(推荐)
&emsp;&emsp;按需加载是指在jackson序列化实体之前，需要手动激活实体中要序列化的懒加载对象，
以查询购物车及其商品列表API`/shoppingCart/{cartId}`为例。
1. 依据JAP中HIbernate的版本，引入`jackson-datatype-hibernate5`工具
```xml
<dependency>
  <groupId>com.fasterxml.jackson.datatype</groupId>
  <artifactId>jackson-datatype-hibernate5</artifactId>
  <version>2.10.0</version>
</dependency>
```
2. 配置序列化规则
```java
@Configuration
public final class Config{
  /**
   * 防止jackson序列化时激活懒加载机制
   * @return 转换器
   */
  @Bean
  public MappingJackson2HttpMessageConverter mappingJackson2HttpMessageConverter() {
      MappingJackson2HttpMessageConverter jsonConverter = new MappingJackson2HttpMessageConverter();
      ObjectMapper objectMapper = jsonConverter.getObjectMapper();
      objectMapper.registerModule(new Hibernate5Module());
      return jsonConverter;
  }
}
```
3. 调用`/shoppingCart/{cartId}`接口，返回我们期望的数据：
```json
{
  "id": 123,
  "goods": [
    {
      "id": 11,
      "name": "华为手机P30",
      "price": 3000.00,
      "count": 10,
      "goodDetails": null
    },
    {
      "id": 12,
      "name": "京东跑步鸡",
      "price": 128,
      "count": 10,
      "goodDetails": null,
    }
  ]
}
```

### 2.2 按非需加载
&emsp;&emsp;按非需加载是指jackson序列化实体时会激活所有关联的懒加载，那么在jackson序列化对象之前，
将不需要的懒加载对象设置为null，以查询购物车及其商品列表API`/shoppingCart/{cartId}`为例。
1. 设置不需要序列化的懒加载对象为null
```java
@Service
public final class ShoppingCartServiceImpl 
    implements ShoppingCartService {
  private ShoppingCartRepository shCaRepository;

  @Autowired
  private ShoppingCartServiceImpl(ShoppingCartRepository shCaRepository) {
    this.shCaRepository = shCaRepository;
  }

  public ShoppingCart getShoppingCart(long cartId) {
    ShoppingCart cart = this.shCaRepository.findById(cartId);
    
    Optional.ofNullable(cart).orElseThrow(
      () -> new NotFoundException(
        String.format("购物车id=%d不存在", cartId)));
    
    // 激活cart中goods懒加载，并将good中的goodDetails懒加载置空
    for(Good good : cart.getGoods()) {
      good.setGoodDetails(null);
    }

    return cart;
  }
}
```
2. 调用`/shoppingCart/{cartId}`接口，返回我们期望的数据：
```json
{
  "id": 123,
  "goods": [
    {
      "id": 11,
      "name": "华为手机P30",
      "price": 3000.00,
      "count": 10,
      "goodDetails": null
    },
    {
      "id": 12,
      "name": "京东跑步鸡",
      "price": 128,
      "count": 10,
      "goodDetails": null,
    }
  ]
}
```
&emsp;&emsp;虽然这种方法在外键少，关联关系简单时可行，但是也会存在引发错误的风险。
1. 风险：由于查询逻辑复杂，导致激活外键时Hibernate Session过期，抛出异常，
此时需要为查询逻辑加一个查询事物`@Transaction`，而spring事物在结束时，如果实体的属性改动了，
会自动更新到数据库，从而导致数据错乱（有点像数据库隔离级别中的不可重复读）。
```java
@Service
public final class ShoppingCartServiceImpl 
    implements ShoppingCartService {
  private ShoppingCartRepository shCaRepository;

  @Autowired
  private ShoppingCartServiceImpl(ShoppingCartRepository shCaRepository) {
    this.shCaRepository = shCaRepository;
  }

  @Transaction
  public ShoppingCart getShoppingCart(long cartId) {
    ShoppingCart cart = this.shCaRepository.findById(cartId);
    
    Optional.ofNullable(cart).orElseThrow(
      () -> new NotFoundException(
        String.format("购物车id=%d不存在", cartId)));
    
    // 激活cart中goods懒加载，并将good中的goodDetails懒加载置空
    for(Good good : cart.getGoods()) {
      good.setGoodDetails(null); // 事物结束时，会认为good中的goodDetails被删除了，并把删除操作更新到数据库
    }

    return cart;
  }
}
```
&emsp;&emsp;此时，已经解决了我们提出的问题，但是通过打印sql语句，我们发下每一次查询Hibernate会像数据库发起多次查询，
也就是`N+1`查询问题。

## 三、sql优化
&emsp;&emsp;为了解决`N+1`问题，优化查询sql，Spring JAP提供`@NamedEntityGraph`与
`@EntityGraph`两组标签来规划实体图，优化后的实例代码如下：
```java
@Data
@Entity
@Table(name = "SHOPPING_CART", schema = "testDB")
@NamedEntityGraph(
    name="shoppingCart.all",
    attributeNodes  = {
        @NamedAttributeNode("goods")
    })
public final class ShoppingCart extends BaseEntity{
  /**
  * 主键
  */
  @Override
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private long id;


  /**
  * 购物车中的商品
  */
  @OneToMany(mappedBy = "shoppingCart", cascade = CascadeType.ALL)
  private Set<Good> goods;

}
```
```java
@Data
@Entity
@Table(name = "GOOD", schema = "testDB")
@NamedEntityGraph(
    name="good.all",
    attributeNodes  = {
        @NamedAttributeNode("shoppingCart"),
        @NamedAttributeNode("goodDetails")
    })
public final class Good extends BaseEntity{
  /**
  * 主键
  */
  @Override
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private long id;

  /**
  * 购物车中商品名称
  */
  private String name;

  /**
  * 购物车中商品数量
  */
  private long count;

  /**
  * 购物车中商品单价
  */
  private double price;


  /**
  * 购物车
  */
  @ManyToOne
  @JoinColumn(referencedColumnName = "id")
  private ShoppingCart shoppingCart;

  /**
  * 商品明细
  */
  @OneToOne(mappedBy = "good", cascade = CascadeType.ALL)
  private GoodDetails goodDetails;

}
```

```java
@Data
@Entity
@Table(name = "GOOD_DETAILS", schema = "testDB")
@NamedEntityGraph(
    name="goodDetails.all",
    attributeNodes  = {
        @NamedAttributeNode("good")
    })
public final class GoodDetails extends BaseEntity{
  /**
  * 主键
  */
  @Override
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private long id;

  /**
  * 商品描述
  */
  private String desc;

  /**
  * 商品图像地址
  */
  private String imgPath;


  @OneToOne
  @JoinColumn(referencedColumnName = "id")
  private Good good;

}
```

```java
public interface ShoppingCartRepository extends
        JpaRepository<ShoppingCart, Long> {
  /**
  * 通过Id查找购物车
  *@param cartId 购物车Id
  *@return 购物车实体
  */        
  @EntityGraph(value="shoppingCart.all",type=EntityGraph.EntityGraphType.FETCH)
  ShoppingCart findById(cartId);
}
```
&emsp;&emsp;再次调用`/shoppingCart/{cartId}`接口，
我们发现Hibernate会组合查询的sql，只向数据库发送1次sql查询请求，从而解决了N+1的问题，
提高了查询性能。    
&emsp;&emsp;`EntityGraph.EntityGraphType`具有**`FETCH`**与**`LOAD`**两个值，
两者的区别在于实体中没有在`@NamedEntityGraph`的attributeNodes中指定的外键对象，**`FETCH`**表明未指定的外键对象采用懒加载，
LOAD表明未指定的外键对象采用自定义方法或者默认方法。
