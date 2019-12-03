## 一、问题背景
&emsp;&emsp;数据库表的实体对象中，往往存在一些状态量，比如（订单状态：待支付、支付中、已支付）：
```java
@Entity
@Table(name = "ORDER", schema = "demo_db")
public final class Order{
  /**
  * 订单状态：1-待支付、2-支付中-已支付
  */
  private String status;
}
```
一般我们会对状态魔数进行静态化处理：
```java
@Entity
@Table(name = "ORDER", schema = "demo_db")
public final class Order{
  /**
  * 订单状态：1-待支付、2-支付中-已支付
  */
  private String status;

  public static final Status {
    public final static final String WAITING_PAY = "1-待支付";
    public final static final String PAYING = "2-支付中";
    public final static final String PAID = "3-已支付";
  }
}
```
但是这样会存在类型安全隐患：

```java
// 另外有一个实体对象商品Good
@Entity
@Table(name = "GOOD", schema = "demo_db")
public final class Good{
  /**
  * 商品类型：1-春装、2-夏装、3、秋装、4-冬装
  */
  private String status;

  public static final Status {
    public final static final String SPRING_DRESS = "1-春装";
    public final static final String SUMMER_DRESS = "2-夏装";
    public final static final String AUTUMN_DRESS = "3-秋装";
    public final static final String WINTER_DRESS = "4-冬装";
  }
}

Order order = new Order();
order.setStatus(Good.Status.SPRING_DRESS); // 其它对象的状态
order.setStatus("hdsajhdajs"); // 魔数
```
于是我们希望能够用带类型约束的枚举`Enum`来代替静态魔数：
```java
@Entity
@Table(name = "ORDER", schema = "demo_db")
public final class Order{
  /**
  * 订单状态：1-待支付、2-支付中-已支付
  */
  private Status status;

  public enum Status {
    WAITING_PAY("1-待支付"),
    PAYING("2-支付中"),
    PAID("3-已支付");

    private String value;

    Status(String value) {
        this.value = value;
    }
    
  }
}
```

&emsp;&emsp;`Enum`解决了类型安全问题，但是带来两个新问题：
1. 如何将`private Status status`在查询和写入数据库表时将`Status`类型转为其原始值的`String`类型？
2. 在`Order`序列化与反序列化时，如何将参数"1-待支付"反序列化为枚举类型`WAITING_PAY`？如果将枚举类型`WAITING_PAY`序列化
为"1-待支付"字符串，返回给调用者？
&emsp;&emsp;本文将给出解决上述问题的通用方案。

## 二、 枚举类型与数据库表原始字段类型的转换
1. 定义`BaseStatus`接口（**枚举类型只能继承接口**）
```java
/**
 * @author xpzsoft
 * @version v1.0.0
 */
public interface BaseStatus<T> {
    /**
    * 获得原始值
    *@return T 数据字段原始类型
    */
    T getValue();
}
```
2. 实现`AttributeConverter`转换器
```java
import javax.persistence.AttributeConverter;
import java.lang.reflect.Method;
import java.lang.reflect.ParameterizedType;

/**
 * @author xpzsoft
 * @version v1.0.0
 */
public class StatusConverter<T extends BaseStatus<E>, E>
        implements AttributeConverter<T, E> {

    /**
    * 将枚举类型BaseStatus转换为数据库字段原始类型
    */
    @Override
    public E convertToDatabaseColumn(T status) {
        return status.getValue();
    }

    /**
    * 将数据库字段原始类型转换为枚举类型
    */
    @SuppressWarnings("unchecked")
    @Override
    public T convertToEntityAttribute(E s) {
        Class t = getGenericClass();
        if (t != null && t.isEnum()) {
            try {
                // 利用enum的静态函数values，将原始类型转为枚举类型
                Method m = t.getMethod("values");
                T []es = (T[])m.invoke(null);
                for (T e : es) {
                    if (e.getValue().equals(s)) {
                        return e;
                    }
                }
            } catch (Exception e) {
                return null;
            }
        }
        return null;
    }

    /**
    * 获取枚举类型泛型的实际类型
    */
    @SuppressWarnings("unchecked")
    private Class<T> getGenericClass() {
        return (Class<T>)((ParameterizedType)getClass().getGenericSuperclass()).getActualTypeArguments()[0];
    }
}
```
3. 更新实体`Order`与`Good`的`Status`
```java
@Entity
@Table(name = "ORDER", schema = "demo_db")
public final class Order{
  /**
  * 订单状态：1-待支付、2-支付中-已支付
  */
  @Convert(converter = OrderStatusConverter.class)
  private Status status;

  public enum Status implements BaseStatus<String> {
    WAITING_PAY("1-待支付"),
    PAYING("2-支付中"),
    PAID("3-已支付");

    private String value;

    Status(String value) {
        this.value = value;
    }

    @Override
    public String getValue() {
      return value;
    }
    
  }

  /**
  * 构建枚举类型转换器
  */
  private static class OrderStatusConverter
      extends StatusConverter<Status, String> {
  }
}
```

```java
@Entity
@Table(name = "GOOD", schema = "demo_db")
public final class Good{
  /**
  * 订单状态：1-待支付、2-支付中-已支付
  */
  @Convert(converter = GoodStatusConverter.class)
  private Status status;

  public enum Status implements BaseStatus<String> {
    SPRING_DRESS("1-春装"),
    SUMMER_DRESS("2-夏装"),
    AUTUMN_DRESS("3-秋装"),
    WINTER_DRESS("4-冬装");

    private String value;

    Status(String value) {
        this.value = value;
    }

    @Override
    public String getValue() {
      return value;
    }
    
  }

  private static class GoodStatusConverter
      extends StatusConverter<Status, String> {
  }
}
```