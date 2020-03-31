# 我与微服务之正确理解JAP（hibernate）的外键映射关系与配置
## 一、问题背景
&emsp;&emsp;基于Spring Cloud微服务开发的过程中，我们一般采用spring data jpa操作数据实体，而数据实体中外键几乎不可避免。
jpa为了方便开发成员管理数据实体以及它们的外键对象，设计了`@OneToOne`、`@ManyToOne`、`@OneToMany`和`@ManyToMany`四种
实体外键对应关系（Mapping Relationship），为了保证数据实体的访问安全，设计了`PERSIST`、`MERGE`、`REFRESH`、`REMOVE`、`DETACH`和`ALL`
六种实体外键级联策略（Cascade Category），为了提高数据访问效率，设计了`LZAY`、`EAGER`、`@EntityGraph`等多种外键实体检索方式（Fetch Type）。
本文将详细介绍对应关系、级联策略与访问效率的使用方法，并尝试分析其原理。
## 二、对应关系（Mapping Relationship）
1. 主表从表的默认加载方式、对懒加载是否支持、是否通过join关联检索

|     | 主表(包含外键)  |  子表(被外键引用)  |
|  ----   | ----  | ---- |
|  @OneToOne | EAGER&#124;true&#124;true | EAGER&#124;false&#124;false |
|  @OneToMany | | LAZY&#124;true&#124;true |
|  @ManyToOne | EAGER&#124;true&#124;true |  |
|  @ManyToMany | LAZY&#124;true&#124;true | LAZY&#124;true&#124;true |

**从表中可以看出，只有在@OneToOne的对应关系中，子表的懒加载无效，且检索时不会采用join关联检索，
即检索n条数据子表记录，jpa会发送n条sql语句去检索对应的主表记录。**
## 三、外键级联策略（Cascade Category）
1. 作用与应用范围

|     | 作用  |  权限  |
|  ----   | ----  | ---- |
| PERSIST | 级联保存外键对象。 如果外键对象为新对象，则插入； 如果外键对象已存在，则更新外键对象 | 对外键对象有插入、更新操作权限 |
| MERGE | 级联保存外键对象。 当且仅当外键对象已存在，则更新外键对象 | 仅对外键对象有更新操作权限 |
| REFRESH |  |  |
| REMOVE | 级联删除外键对象 | 仅对外键对象有删除操作权限 |
| DETACH |  |  |
| ALL | 上述5中操作集合 | 上述5中操作权限集合 |

## 四、检索方式（Fetch Type）
详情见[我与微服务之正确理解Hibernate LAZY、EAGER、JAP @EntityGraph的关系](我与微服务之正确理解Hibernate LAZY、EAGER、JAP @EntityGraph的关系.md)
