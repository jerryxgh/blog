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
mybatis-plus支持4种主键策略，

# 更新-null更新策略
mybatis-plus默认只更新非null字段，之前有类似updateAllColumn的方法，但是在v2.1.6版本之后被移除了，现在可以通过手工拼接sql的方式做到更新字段为null。

# 查询-复杂sql和分页


# 项目活跃度
通用Mapper的绝大部分代码是作者一个人贡献的，因此项目的整体性更好，从github代码提交来看一直在活跃维护中；mybatis-plus是由多人合作写成，近期也在快速变化中，从项目注释角度看略显混乱，但是非常活跃。

# 总结
通用Mapper主要由一人写成，项目相对比较稳定，能够兼容JDK1.6+，此外也能用库支持1.8+的语法，而mybatis-plus由多人合作写成，项目活跃度较高，但是只支持java1.8+。

# 参考资料
1. [Mybatis代码生成器](http://www.mybatis.org/generator/)
2. [通用Mapper](https://github.com/abel533/Mapper)
3. [mybatis-plus](https://github.com/baomidou/mybatis-plus)
