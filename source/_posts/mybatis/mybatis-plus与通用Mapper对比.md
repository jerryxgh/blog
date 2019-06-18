---
title: mybatis-plus与通用Mapper对比
date: 2019-06-12 00:06:46
categories:
- mybatis
tags:
- 原创
- java
- mybatis-plus
- 通用Mapper
- tk mapper
---
在使用Mybatis的时候，有两种写CRUD的方式，一种是使用xml文件或者Java的注解手写sql，一种是用Mybatis的代码生成器(MBG)自动生成，这两种笔者都经历过。手写的方式有个很大缺点就是效率太低，CRUD的代码其实绝大部分是重复的，不同的往往只是表名而已，因此手写往往就是copy&paste，这个过程相当枯燥而且容易出错；代码生成器在效率上好多了，MBG能够帮助生成功能全面的Mapper或者DAO，但是问题也很明显，就是每次新增字段的时候重新生成代码，但是老代码可能被定制化修改过了，比如增加了一些非CRUD的接口，这时就需要做代码合并，对后期的维护很不友好。

回头看，无论是手写sql还是MBG代码生成，实际产出的代码，在模式上有大量的重复，这种重复是导致问题的根源，笔者在遇到通用Mapper和Mybatis-plus之前，也一直在苦苦寻觅解决方案，直到在一次偶然的交流中听到有通用Mapper这种库，CRUD的问题犹如拨云雾而见青天，原来还有这么优雅的解决方案！笔者在此再次深深感谢通用Mapper和Mybatis-plus的作者提供这么优秀的类库。实际项目中笔者使用过这两个类库，本文是对这两个优秀的mybatis增强库进行比较和总结，不当之处请大家拍砖。

# jdk兼容性
通用Mapper支持Java1.6+，使用weekend库可以支持Java1.8+的语法，而mybatis-plus只支持Java1.8+。此外两者都支持除了本身提供的CRUD接口外，用户通过Mybatis注解或者XML扩展Mapper(Dao)能力。

# 创建-主键生成策略
通用Mapper的主键策略，核心是KeySql注解，直接看KeySql类的代码就能理解：
1. 如果数据库支持字段自增并且JDBC驱动支持`getGeneratedKeys`，推荐使用（例如MySQL）。配置样例：
```Java
@Id
@KeySql(useGeneratedKeys = true)
private Long id;
// 或者都使用JPA注解：
@Id
@GeneratedValue(generator = "JDBC")
private Long id;
```
2. 如果数据库不支持`getGeneratedKeys`，但是支持自增，可以用下面的方式：
```Java
@Id
//DEFAULT 需要配合 IDENTITY 参数（ORDER默认AFTER）
@KeySql(dialect = IdentityDialect.DEFAULT)
private Integer id;
//建议直接指定数据库
@Id
@KeySql(dialect = IdentityDialect.MYSQL)
private Integer id;


// 目前通用Mapper支持了以下的数据库：
DB2("VALUES IDENTITY_VAL_LOCAL()"),
MYSQL("SELECT LAST_INSERT_ID()"),
SQLSERVER("SELECT SCOPE_IDENTITY()"),
CLOUDSCAPE("VALUES IDENTITY_VAL_LOCAL()"),
DERBY("VALUES IDENTITY_VAL_LOCAL()"),
HSQLDB("CALL IDENTITY()"),
SYBASE("SELECT @@IDENTITY"),
DB2_MF("SELECT IDENTITY_VAL_LOCAL() FROM SYSIBM.SYSDUMMY1"),
INFORMIX("select dbinfo('sqlca.sqlerrd1') from systables where tabid=1"),
```
3. 以上两种都不支持的时候，可以显示指定获取id的sql，例如：
```Java
@Id
@KeySql(sql = "select SEQ_ID.nextval from dual", order = ORDER.BEFORE)
private Integer id;
```

而mybatis-plus用下面的方式支持5种主键策略，分别如下：
1. IdType.AUTO
数据库ID自增，当使用这种策略时，如果用户手工设置了`主键`的值，mybatis-plus不会使用，而是用数据库自增的方式生成，也就是在写入记录的时候用户无法手工指定`主键`值。
2. IdType.NONE
IdType.NONE是`主键`的默认类型，如果不显示指定`主键`的类型，那么就是NONE类型，NONE类型的特点是用户没有指定主键的值，使用下面的IdType.ID_WORKER策略生成主键，否则使用用户指定的主键值。
3. IdType.INPUT
IdType.INPUT使用用户设定的主键值，包括null。在mysql的场景，如果主键的值是null，使用自增。
4. ID_WORKER和ID_WORKER_STR
基于雪花（snowflake）生成sequence，可以生成64bit的Long类型主键，其中ID_WORKER_STR生成字符类型的主键，忽略用户的输入，始终用算法生成主键，大致算法如下：
```java
// 时间戳部分 | 数据中心部分 | 机器标识部分 | 序列号部分
return ((timestamp - twepoch) << timestampLeftShift)
    | (datacenterId << datacenterIdShift)
    | (workerId << workerIdShift)
    | sequence;
```

5. uuid
使用uuid生成主键，忽略用户的输入，始终用算法生成主键，主键类型必须是字符串。

mybatis-plus虽然主键策略有5种，但最实用的是AUTO和INPUT，INPUT应该是最常用的类型，尽量不要使用NONE。相比mybatis-plus，通用Mapper没有进一步封装外部生成主键的机制，而是集中在数据库自己生成主键上，如果使用外部方式生成主键，用户可以调用类似Sequence生成主键，显示设置到对象里，之后通用Mapper会自动跳过主键生成，直接使用用户设置的主键。整体上通用Mapper的主键方式设计更合理，使用更简单。

# 更新-null更新策略
mybatis-plus默认只更新非null字段，之前有类似updateAllColumn的方法，但是在v2.1.6版本之后被移除了，现在可以通过手工拼接sql的方式做到更新字段为null。而通用Mapper的默认update方法就会把null的字段对应的数据库字段更新成null，除非用updateXxxSelective方法，只更新有值的字段。


# 项目活跃度
通用Mapper的绝大部分代码是作者一个人贡献的，因此项目的整体性更好，从github代码提交来看一直在活跃维护中；mybatis-plus是由多人合作写成，近期也在快速变化中，从项目注释角度看略显混乱，但是非常活跃。

# 注解选择
通用Mapper尽量复用JPA的注解规范，在必须的时候提供自己定义的注解以增强表达简化配置，mybatis-plus基本上都是自定义的注解，因此相对对于熟悉JPA的同学，更容易学习通用Mapper。

# 代码量
包括单元测试代码情况下，通用Mapper的java文件数量是329，去除测试代码后的文件数量是272，去除测试代码后的代码行数是23401，也就是只有两万多行代码，建议彻底看完，还是很精简的。

而mybatis-plus在包括包括单元测试代码情况下，java文件数量是411，去除测试代码后的文件数量是361，去除测试代码后的代码行数是33797，代码相比通用Mapper多了接近一半。

从代码量的角度，如果功能相同或者相近的情况下，当然是越少越好，通用Mapper的代码相比mybatis-plus的代码少了接近一半，Java文件的数量也少了接近一半。 这是我的统计方法，不一定精确，但是有参考价值：
```sh
# 包括测试文件的java文件数量
find . -name '*.java' | wc -l
# 不包括测试文件的java文件数量
find . -name '*.java' | grep -v "Test" | wc -l
# 不包括测试文件的java代码行数
find . -name '*.java' | grep -v "Test" | xargs wc -l
```

# 总结
通用Mapper主要由一人写成，项目相对比较稳定，能够兼容JDK1.6+，此外也能用库支持1.8+的语法，而mybatis-plus由多人合作写成，项目活跃度较高，但是只支持java1.8+；此外通用Mapper的代码量较少，在主键策略、注解选择、接口设计上感觉更合理，不清楚为什么在外部的反响反而并不如mybatis-plus，这也算是一桩怪事吧。

# 参考资料
1. [Mybatis代码生成器](http://www.mybatis.org/generator/)
2. [通用Mapper](https://github.com/abel533/Mapper)
3. [mybatis-plus](https://github.com/baomidou/mybatis-plus)
4. [分布式高效ID生产黑科技](https://gitee.com/yu120/sequence)
