# Domain
## 1. Oracle批量插入数据
jpa插入数据逻辑：  
1. 批量插入n条数据（n个对象）
2. 从sequence（序列）中获取allocationSize（allocationSize >= n， 默认为50）个主键
```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "seq")
@SequenceGenerator(name = "seq", sequenceName = "SYS_ID", allocationSize = n)
@Column(name = "MYID", nullable = false, precision = 0)
```  
注意，如果allocationSize < n，不能保证序列取出的主键值的唯一性，可能会抛出异常：
```java
A different object with the same identifier value was already associated with the session 
```
3. allocationSize设置为1，满足批量插入要求
4. 为每个插入对象赋值主键
5. 将对象从内存持久化到数据库

## 2. 枚举类型与数据类型的转换
```java
@Entity
@Table(name = "Z_APP_ASY", schema = "BAMTRI_MES")
public class FirAppAsyEntity {
    private long id;
    private String name;
    private String param;
    private Status status;

    /.../

    public enum Status {
        /**
         * 等待执行
         */
        RUNNING(2L),
        /**
         * 执行成功
         */
        SUCCESS(1L),

        /**
         * 执行失败
         */
        FAILED(0L);

        private long value;

        Status(long value) {
            this.value = value;
        }

        public long getValue() {
            return value;
        }

        public static Status getTypeFromName(long status){
            for (Status type : Status.values()) {
                if (type.getValue() == status){
                    return type;
                }
            }
            return null;
        }
    }

    

    @Converter(autoApply = true)
    public static class StatusConverter implements AttributeConverter<Status, Long> {
        @Override
        public Long convertToDatabaseColumn(Status status) {
            return status.getValue();
        }

        @Override
        public Status convertToEntityAttribute(Long aByte) {
            return Status.getTypeFromName(aByte);
        }
    }
}
```
注意，Oracle中数字类型为Number，如果枚举类型AttributeConverter<Status, Long>改为AttributeConverter<Status, Byte>，则Spring将Number视为Byte[]，会导致数据操作失败。