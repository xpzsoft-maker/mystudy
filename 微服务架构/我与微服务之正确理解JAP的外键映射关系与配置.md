# 我与微服务之正确理解JAP的外键映射关系与配置
## 一、问题背景
&emsp;&emsp;基于Spring Cloud微服务开发的过程中，我们一般采用spring data jpa操作数据实体，而数据实体中外键几乎不可避免。
jpa为了方便开发成员管理数据实体以及它们的外键对象，设计了`@OneToOne`、`@ManyToOne`、`@OneToMany`和`@ManyToMany`四种
实体外键对应关系（Mapping Relationship），为了保证数据实体的访问安全，设计了`PERSIST`、`MERGE`、`REFRESH`、`REMOVE`、`DETACH`和`ALL`
六种实体外键级联策略（Cascade Category），为了提高数据访问效率，设计了`LZAY`、`EAGER`、@EntityGraph等多种外键实体检索方式（Fetch Type）。本文将详细介绍
对应关系、级联策略与访问效率的使用方法，并尝试分析其原理。
## 二、对应关系（Mapping Relationship）
1. @OneToOne
&emsp;&emsp;众所周知，一般我国公民每人只有一个有效身份证（原则上），也就是说公民与合法身份证是一对一的关系，而身份证属于公民，
所以我们设计数据表时，**正确的主从关系应该为公民表是主表，身份证表是从表**，换成jpa实体对象，应该为：
```java
@Entity
@Table("citizen")
public final class CitizenEntity{
	@Id
	private int id;

	/**
	* 年龄
	*/
	private int age;

	/*
	* 身份证，从表对象
	*/
	@OneToOne(fetch=FetchType.LZAY)
	@JoinColum(name="idcard_id")
	private IdCardEntity idCard;
}

@Entity
@Table("id_card")
public final class IdCardEntity {
	@Id
	private int id;

	/**
	* 身份证号码
	*/
	private String no;

	/**
	* 姓名
	*/
	private String name;

	/**
	* 性别
	*/
	private byte sex;

	/**
	* 地址
	*/
	private String address;

	/**
	* 发证机关
	*/
	private String issuingAuthority;

	/**
	* 公民，主表对象
	*/
	@OneToOne(mappedBy="idCard")
	private CitizenEntity citizen;
}
```
此时，**外键定义在公民表中，fetch=FetchType.LZAY此时起作用，因为jpa检索公民表时，能够根据字段`idcard_id`中的值是否为空，
来判断是否生成代理对象。如果`idcard_id`为空，则不生成代理对象，属性`idCard=null`；如果`idcard_id`不为空，则说明
有外键对象，则生成代理对象，`idCard=jpaProxy`，等待外部程序触发懒加载。**
&emsp;&emsp;但有时候，可能不小心将外键设置到身份证对象中，**即身份证表成为了主表，公民表成为了从表，从而在逻辑上错误
地设置了主从关系**，换成jpa实体对象后，其关系为：
```java
@Entity
@Table("citizen")
public final class CitizenEntity{
	@Id
	private int id;

	/**
	* 年龄
	*/
	private int age;

	/*
	* 身份证，从表对象
	*/
	@OneToOne(fetch=FetchType.LZAY, mappedBy="citizen")
	private IdCardEntity idCard;
}

@Entity
@Table("id_card")
public final class IdCardEntity {
	@Id
	private int id;

	/**
	* 身份证号码
	*/
	private String no;

	/**
	* 姓名
	*/
	private String name;

	/**
	* 性别
	*/
	private byte sex;

	/**
	* 地址
	*/
	private String address;

	/**
	* 发证机关
	*/
	private String issuingAuthority;

	/**
	* 公民，主表对象
	*/
	@OneToOne
	@JoinColum(name="citizen_id")
	private CitizenEntity citizen;
}
```
此时，通过jpa查找公民信息时，jpa报错，大概意思为没有可用的session来查询IdCardEntity信息，即CitizenEntity中的fetch=FetchType.LZAY设置错误（不起作用），
当设置为fetch=FetchType.EAGER时正常运行（或者设置@EntityGraph），这就破坏了jap的加载优化机制，即外键对象变成了必须加载项，而且还是分两sql请求完成加载，
第1次是查询出公民信息，第2次根据公民的id，查询身份证信息。为何会这样？原因就是主从逻辑的错误设定，身份证成为了主表，外键关系建立在身份证表中，导致每次查找公民
信息时，jpa拿不到外键信息（外键字段在身份证表中），并不知道外键是否存在，是否有值，因次必须要去身份证表中做检索，从而导致上述问题。
（疑惑：按照逻辑来说，jpa在这查询从表对象时，也可以启用懒加载，即客户程序调用从表对象中的主表对象时再去做检索，这似乎也没什么不妥呀，为何不这样设计呢？
我觉得很有可能与空值比较有关，在主表实体中，不用检索从表实体，就能判断从表实体是否为null，而从表就不行，如果要判断一个从表实体中的主表实体是否为null，必须要去检索主表才能知道，
即if(citizen.getIdCard() == null)这个条件，必须要进行检索才能判断，否则从表无法判断，再加上FetchType.LZAY中，citizen.getIdCard()不会触发sql检索，
citizen.getIdCard().getId()才能触发sql检索，导致空值判断逻辑冲突）
