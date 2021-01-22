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

  /**
  * 定义枚举类型
  */
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
  * 订单状态：1-待支付、2-支付中、3-已支付
  */
  @Convert(converter = OrderStatusConverter.class)
  private Status status;

  /**
  * 定义枚举类型
  */
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
  private final static class OrderStatusConverter
      extends StatusConverter<Status, String> {
  }
}
```

```java
@Entity
@Table(name = "GOOD", schema = "demo_db")
public final class Good{
  /**
  * 订单状态：1-春装、2-夏装、3-秋装、4-冬装
  */
  @Convert(converter = GoodStatusConverter.class)
  private Status status;

  /**
  * 定义枚举类型
  */
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

  /**
  * 构建枚举类型转换器
  */
  private final static class GoodStatusConverter
      extends StatusConverter<Status, String> {
  }
}
```
```java
Order order = new Order();
order.setStatus(Good.Status.SPRING_DRESS); // 编译报错
order.setStatus("hdsajhdajs"); // 报错编译
order.setStatus(Order.Status.WAITING_PAY); //正确
```
&emsp;&emsp;到此，我们解决了状态量的类型安全问题，但是实体json序列化后的数据如下：
```json
// Order实体序列化，我们不希望的数据
{
  "status": "WAITING_PAY"
}
```
很显然，在微服务设计API接口的时候，这不是我们想要的结果，我们希望API接口返回的数据是有意义的，数据库中本地化的实际数据：
```json
// Order实体序列化，我们希望的数据
{
  "status": "1-待支付"
}
```
## 三、枚举类型序列化为原始数据，原始数据反序列化为枚举类型(jackson)
1. 构建枚举类型序列化为原始类型的基类
```java
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.databind.JsonSerializer;
import com.fasterxml.jackson.databind.SerializerProvider;

import java.io.IOException;

/**
 * 此处以BaseStatus<String>为例，枚举类型序列化为String类型
 * @author xpzsoft
 * @version v1.0.0
 */
public class StatusSerializer<T extends BaseStatus<String>> extends JsonSerializer<T> {
    @Override
    public void serialize(
            T status,
            JsonGenerator jsonGenerator,
            SerializerProvider serializerProvider) throws IOException {
        jsonGenerator.writeString(status.getValue());
    }
}
```
2. 构建原始类型反序列化为枚举类型的基类
```java
import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.databind.DeserializationContext;
import com.fasterxml.jackson.databind.JsonDeserializer;
import com.fasterxml.jackson.databind.JsonNode;

import java.io.IOException;
import java.lang.reflect.Method;
import java.lang.reflect.ParameterizedType;

/**
 * 此处以BaseStatus<String>为例，String类型反序列化为枚举类型
 * @author xpzsoft
 * @version v1.0.0
 */
public class StatusDeserializer <T extends BaseStatus<String>> extends JsonDeserializer<T> {
    @SuppressWarnings("unchecked")
    @Override
    public T  deserialize(
            JsonParser jsonParser,
            DeserializationContext deserializationContext) throws IOException {
        JsonNode node = jsonParser.readValueAs(JsonNode.class);
        String statusStr = node.asText();
        Class t = getGenericClass();
        if (t != null) {
            try {
                Method m = t.getMethod("values");
                T []es = (T[]) m.invoke(null);
                for (T e : es) {
                    if (e.getValue().equals(statusStr)) {
                        return e;
                    }
                }
                m = t.getMethod("valueOf", String.class);
                return (T) m.invoke(null, statusStr);
            } catch (Exception e) {
                return null;
            }
        }
        return null;
    }

    @SuppressWarnings("unchecked")
    private Class<T> getGenericClass() {
        Object [] a = ((ParameterizedType)getClass().getGenericSuperclass()).getActualTypeArguments();
        return (Class<T>)((ParameterizedType)getClass().getGenericSuperclass()).getActualTypeArguments()[0];
    }
}
```
3. 更新实体`Order`与`Good`
```java
@Entity
@Table(name = "ORDER", schema = "demo_db")
public final class Order{
  /**
  * 订单状态：1-待支付、2-支付中、3-已支付
  */
  @JsonSerialize(using = OrderStatusSerializer.class)
  @Convert(converter = OrderStatusConverter.class)
  private Status status;

  /**
  * 定义枚举类型
  */
  @JsonDeserialize(using = OrderStatusDeserializer.class)
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
  private final static class OrderStatusConverter
      extends StatusConverter<Status, String> {
  }

  /**
  * 构建反序列化工具
  */
  private static final class OrderStatusDeserializer
      extends StatusDeserializer<Status> {
  }

  /**
  * 构建序列化工具
  */
  private final static class OrderStatusSerializer
      extends StatusSerializer<Status> {
  }
}
```

```java
@Entity
@Table(name = "GOOD", schema = "demo_db")
public final class Good{
  /**
  * 订单状态：1-春装、2-夏装、3-秋装、4-冬装
  */
  @JsonSerialize(using = GoodStatusSerializer.class)
  @Convert(converter = GoodStatusConverter.class)
  private Status status;

  /**
  * 定义枚举类型
  */
  @JsonDeserialize(using = GoodStatusDeserializer.class)
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

  /**
  * 构建枚举类型转换器
  */
  private final static class GoodStatusConverter
      extends StatusConverter<Status, String> {
  }

  /**
  * 构建反序列化工具
  */
  private static final class GoodStatusDeserializer
      extends StatusDeserializer<Status> {
  }

  /**
  * 构建序列化工具
  */
  private final static class GoodStatusSerializer
      extends StatusSerializer<Status> {
  }
}
```

4. 测试

+ 序列化

```java
Order order = new Order(Order.Status.WAITING_PAY);
// objectMapper为jackson工具对象
string jsondata = objectMapper.writeValueAsString(order);
```
输入出为：
```json
{
  "status": "1-待支付"
}
```

+ 反序列化

```java
String jsondata = "{\"status\": \"1-待支付\"}"
// objectMapper为jackson工具对象
Order order = objectMapper.readValue(jsondata, Order.class);
```

## 四、待优化
&emsp;&emsp;细心的同学可能会发现，`public class StatusSerializer<T extends BaseStatus<String>> extends JsonSerializer<T>`
与`public class StatusDeserializer <T extends BaseStatus<String>> extends JsonDeserializer<T>`缩小了应用范围，都是针对原始数据
为`String`的枚举类型展开，如果原始类型为Integer，则需要重写`public class StatusSerializer<T extends BaseStatus<Integer>> extends JsonSerializer<T>`
与`public class StatusDeserializer <T extends BaseStatus<Integer>> extends JsonDeserializer<T>`，这样感觉比较冗余，最好的方式则是使用泛型，
`public class StatusSerializer<T extends BaseStatus<E>, E> extends JsonSerializer<T>`
与`public class StatusDeserializer <T extends BaseStatus<E>, E> extends JsonDeserializer<T>`，不过针对E具体类型，序列化与反序列化的过程稍有差别，
需要拿到运行时E的具体类型再做转换，后续再解决这个问题。