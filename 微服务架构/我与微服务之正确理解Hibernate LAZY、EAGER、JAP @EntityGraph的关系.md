## 一、问题背景

&emsp;&emsp;`Spring Data JPA`默认采用`Hibernate`作为`ORM`框架，`Hibernate`在处理关系实体的外键关联时，
有`LAZY`与`EAGER`两种方式，`LAZY`虽然解决了按需加载的问题，但同时也带来了`N+1`访问问题，增加了数据库的压力，
于是`Spring Data JPA`提出了`@EntityGraph`注解，解决N+1问题。本文将说明`LAZY`、`EAGER`与`@EntityGraph`的关系及应用场景。

## 二、关系
&emsp;&emsp;为了表达更清楚，本文接下来的例子均将以订单`Order`和商品`Good`和商品明细`GoodDetails`为例进行说明。
设定三者数据表如下表所示：   
ORDER表
| id | code |
| :--: | :--: |
| 1 | abc |
| 2 | def |

GOOD表
| id | name | order_id |
| :--: | :--: | :--: |
| 1 | 歼20模型 | 1 |
| 2 | 华为P30 | 1 |
| 3 | 京东跑步鸡 | 2 |

GOOD_DETAILS表
| id | count | good_id |
| :--: | :--: | :--: |
| 1 | 100 | 1 |
| 2 | 200 | 2 |
| 3 | 100000 | 3 |




1. `LAZY`与`EAGER`

&emsp;&emsp;`LAZY`与`EAGER`是`Hibernate`在处理关系实体的外键关联时的两种方式。

+ `EAGER`实时加载查询，实体`a`与实体`b`有外键关联，那么查询`a`时，自动`join`查询出`b`，
同理查询`b`时自动join查询出`a`
&emsp;&emsp;以`Order`和`Good`为例：
```java
@Entity
@Table(name = "ORDER", schema = "demo_db")
public final class Order{
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private long id;

  /**
  * 订单编码
  */
  private String code;

  @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, fetch=FetchType.EAGER)
  private Set<Good> goods;
}
```

```java
@Entity
@Table(name = "GOOD", schema = "demo_db")
public final class Good{
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private long id;

  /**
  * 商品名称
  */
  private String name;

  @ManyToOne
  @JoinColumn(referencedColumnName = "id")
  private Order order;
}
```
那么，当我们查询订单时，自动将关联的商品也查出来。
```java
public interface OrderRepository extends
        JpaRepository<Order, Long> {
  Order findById(orderId);
}

...

Order order = orderRepository.findById(1);
String jsondata = objectMapper.writeValueAsString(order);

```
`jsondata`输出如下所示：
```json
{
  "id": 1,
  "code": "abc",
  "goods": [
    {
      "id": 1,
      "name": "歼20模型"
    },
    {
      "id": 2,
      "name": "华为P30"
    }
  ]
}
```
但很多时候，我们的操作习惯是先查看订单列表，再选中列表中的某个订单，再点进去查看订单详细信息，
如果在查询订单列表时，以`EAGER`方式将订单的所有详细信息也查出来，显然是冗余的查询操作，
在订单与商品数量巨大的情况下，严重影响性能。为了解决这个问题，Hibernate退出了`LAZY`延时加载模式。

+ `LAZY`延时加载查询（又叫懒加载查询或者按需加载查询），实体`a`与实体`b`有外键关联，那么查询`a`时，不会自动查询出`b`，
而是生成一个针对`b`的代理`proxy`，如果业务中需要用到`b`，需要在程序中调用`proxy`，触发对`b`的查询，
如果先查询`b`再查询`a`也是上述逻辑
&emsp;&emsp;以`Order`和`Good`为例：
```java
@Entity
@Table(name = "ORDER", schema = "demo_db")
public final class Order{
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private long id;

  /**
  * 订单编码
  */
  private String code;

  @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, fetch=FetchType.LAZY)
  private Set<Good> goods;
}
```

```java
@Entity
@Table(name = "GOOD", schema = "demo_db")
public final class Good{
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private long id;

  /**
  * 商品名称
  */
  private String name;

  @ManyToOne
  @JoinColumn(referencedColumnName = "id")
  private Order order;
}
```
那么，当我们查询订单时，商品不会被查询出来。
```java
public interface OrderRepository extends
        JpaRepository<Order, Long> {
  Order findById(orderId);
}

...

Order order = orderRepository.findById(1);
String jsondata = objectMapper.writeValueAsString(order);

```
`jsondata`输出如下所示([查看jackson与JAP懒加载冲突解决方案](./我与微服务之jackson与JPA懒加载冲突解决方案.md))：
```json
{
  "id": 1,
  "code": "abc"
}
```
如果需要查询商品，则需要调用`proxy`，手动触发对`b`的查询：
```java
...

Order order = orderRepository.findById(1);

// 调用`proxy`，手动触发对`b`的查询
order.getGoods().size();

String jsondata = objectMapper.writeValueAsString(order);

```

`jsondata`输出如下所示：
```json
{
  "id": 1,
  "code": "abc",
  "goods": [
    {
      "id": 1,
      "name": "歼20模型"
    },
    {
      "id": 2,
      "name": "华为P30"
    }
  ]
}
```
延时加载查询解决了按需加载的问题，使得查询更加灵活，但是带来N+1问题。

2. N+1问题
&emsp;&emsp;以`Order`和`Good`为例，当需要一次查询多个订单及其所含商品时：
```java

List<Order> orders = orderRepository.findAll();

// 调用`proxy`，手动触发对`b`的查询
for (Order order : orders) {
  order.getGoods().size();
}

```
此时`Hibernate`向数据库发起了多次查询请求：
```sql
# 第1条
select * from ORDER
# 第2条
select * from GOOD where order_id = 1
# 第3条
select * from GOOD where order_id = 2
```
很明显，`1`条`join`查询语句能完成的查询操作，变成了`2+1`条（这里的`N`就是关联外键的实体数量，此处`N`=2），
增加了数据库的压力，影响性能。   
`Hibernate`陷入了尴尬的境地，**EAGER不存在N+1的问题，但是太死板，不能按需加载，LAZY虽然灵活，能够按需加载，但是又有N+1问题**，
为了缓解这个矛盾，`Spring Data JPA`推出了升级版本的`EAGER`，即`@EntityGraph`。

3. @EntityGraph