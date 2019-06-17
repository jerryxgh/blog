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

回头看，无论是手写sql还是MBG代码生成，实际产出的代码，在模式上有大量的重复，这种重复是导致问题的根源，笔者在遇到通用Mapper和Mybatis-plus之前，也一直在苦苦寻觅解决方案，直到在一次偶然的交流中听到有通用Mapper这种库，CRUD的问题犹如拨云雾而见青天，原来还有这么优雅的解决方案！笔者在此感谢通用Mapper和Mybatis-plus的作者提供这么优秀的类库。笔者在实际项目中使用过这两个类库，结合笔者的观察，对这两个优秀的mybatis增强库进行比较，希望对大家的选型有帮助。

# jdk兼容性
通用Mapper支持Java1.6+，使用weekend库可以支持Java8+的语法，而mybatis-plus只支持1.8+。

# 创建-ID主键生成策略
mybatis-plus支持5种主键策略，分别如下：
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

mybatis-plus虽然主键策略有5种，但最实用的是AUTO和INPUT，INPUT应该是最常用的类型，绝对不能是NONE。

# 更新-null更新策略
mybatis-plus默认只更新非null字段，之前有类似updateAllColumn的方法，但是在v2.1.6版本之后被移除了，现在可以通过手工拼接sql的方式做到更新字段为null。

# 查询-复杂sql和分页


# 项目活跃度
通用Mapper的绝大部分代码是作者一个人贡献的，因此项目的整体性更好，从github代码提交来看一直在活跃维护中；mybatis-plus是由多人合作写成，近期也在快速变化中，从项目注释角度看略显混乱，但是非常活跃。

# 代码量
包括单元测试代码情况下，通用Mapper的java文件数量是329，去除测试代码后的文件数量是272，去除测试代码后的代码行数是23401，也就是只有两万行代码，建议彻底看完，还是很精简的。

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
通用Mapper主要由一人写成，项目相对比较稳定，能够兼容JDK1.6+，此外也能用库支持1.8+的语法，而mybatis-plus由多人合作写成，项目活跃度较高，但是只支持java1.8+。

# 参考资料
1. [Mybatis代码生成器](http://www.mybatis.org/generator/)
2. [通用Mapper](https://github.com/abel533/Mapper)
3. [mybatis-plus](https://github.com/baomidou/mybatis-plus)
4. [分布式高效ID生产黑科技](https://gitee.com/yu120/sequence)
