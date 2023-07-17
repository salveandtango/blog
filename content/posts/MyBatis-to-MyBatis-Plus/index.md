---
title: 如果可以，我希望你也了解下MyBatis-Plus
date: 2021-07-01 12:06:55
categories: ["technical"]
---

## 前言

 谈一谈 MyBatis-Plus，和我使用它的感受。[MyBatis-Plus](https://baomidou.com/guide/) 是国人在 MyBatis 的基础上开发的一款增强工具。它能够很好地简化开发和提升效率，最初的 1.0 版本是 2016 年 7 月份发行的，截至目前为止版本号已经来到了 3.4.3。我从 19 年开始使用 MyBatis-Plus 代替 Mybatis，它帮我省去了非常多的重复操作。

_（注：以下操作全为在 SpringBoot 中进行）_

## 使用

相对于 Mybatis，Plus 基本不用编写 XML 文件进行配置，一切都可以通过 yml 进行配置，而其他插件也可以通过配置类的形式实现（例如分页配置只需通过向 MybatisPlusInterceptor 注入 PaginationInnerInterceptor 即可），具体操作可参考[官方文档](https://mybatis.plus/guide/interceptor.html#mybatisplusinterceptor)。下面介绍一下我常用的功能。

### MyBatis-Plus-Generator

> AutoGenerator 是 MyBatis-Plus 的代码生成器，通过 AutoGenerator 可以快速生成 Entity、Mapper、Mapper XML、Service、Controller 等各个模块的代码，极大的提升了开发效率。

在早期版本时使用这个插件会遇到一些依赖冲突方面的问题，不过目前随着版本更新这个问题也解决了，推荐使用最新的版本。MyBatis-Plus-Generator 不仅可以生成实体类和 Mapper.xml 文件，还可以生成对应的 Mapper 接口、Service 接口、Service 实现类和 Controller。通过修改配置类和自定义配置，还可以对生成的文件进行其他不同的操作，例如指定生成目录、名称等，具体使用方法参考[官方文档](https://mybatis.plus/guide/generator.html#%E4%BD%BF%E7%94%A8%E6%95%99%E7%A8%8B)。

### CURD 接口与条件构造器

MyBatis-Plus 提供的 CURD 接口非常丰富，它提供的[IService 类](https://gitee.com/baomidou/mybatis-plus/blob/3.0/mybatis-plus-extension/src/main/java/com/baomidou/mybatisplus/extension/service/IService.java)除了包含基础的增删改查之外，还有一些批量操作、分页查询等。只需将 Service 接口继承 IService 即可使用这些方法，通过代码生成器生成的接口也遵循这一规则。这些方法需要配合[条件构造器](https://mybatis.plus/guide/wrapper.html#abstractwrapper)使用，配置不同的条件构造器能够让你使用 IService 接口完成不同的 SQL 操作，在没有特殊 SQL 查询的情况下，MyBatis-Plus 提供的接口涵盖了大部分增删改查的业务需求，着实让我这个做后台的省了不少事。例如下列代码查询了 User 表中 user_name 字段为 Fixed 的数据，相当于<font color="skyblue">select \* from user where user_name = 'Fixed'</font> 的 SQL 语句：

```java
// 假设user表的实体类为User，Service接口名为userService，
QueryWapper<User> queryWapper = new QueryWapper<>();
queryWapper.eq("user_name","Fixed");
userService.getOne(queryWapper);
```

值得一提的是，MyBatis-Plus 的分页查询功能非常灵活，通过配置它的条件构造器，可以在不用编写 sql 语句的情况下进行分页条件查询，而在需要编写特殊 SQL 查询时，也可以通过在 Mapper 接口中加入 Page 参数即可完成分页功能。例如：

```java
/**
  * 获取用户列表
  *
  * @param userName 用户名
  * @param page     page对象
  * @return iPage
  */
IPage<SysUser> selectUserList(String userName, Page<SysUser> page);
```

```sql
<select id="selectUserList" resultMap="userWithRole">
    SELECT su.id,
    su.user_name,
    su.create_time,
    su.gender,
    su.update_time,
    sr.id role_id,
    sr.role_name,
    sa.id attribution_id,
    sa.attribution_name
    FROM sys_user su
    LEFT JOIN sys_user_role sur ON su.id = sur.user_id
    LEFT JOIN sys_role sr ON sr.id = sur.role_id
    LEFT JOIN sys_attribution sa on sr.attribution_id = sa.id
    <if test="userName!=null and userName!=''">
        WHERE su.user_name = #{userName}
    </if>
</select>
```

### MybaitsX 快速开发插件

> MybatisX 是一款基于 IDEA 的快速开发插件，为效率而生。
>
> 安装方法：打开 IDEA，进入 File -> Settings -> Plugins -> Browse Repositories，输入 `mybatisx` 搜索并安装。

[GitHub 地址](https://gitee.com/baomidou/MybatisX)，这款插件可以让你从 Mapper 定义接口后在 xml 文件自动生成 SQL 代码，虽然非常方便但是我使用的频率并不高，更多时候它的作用是帮助我定位 Mapper 接口与 XML 文件中 SQL 代码的位置，并且可以检测 XML 中 SQL 返回值与 Mapper 接口的返回类型是否对应，避免程序员在修改 Mapper 或者 SQL 后忘记修改另一处时出错。

## 总结

上述这些操作如果要使用 MyBatis 来完成，则不可避免的要花不少时间来写 SQL 文件和配置分页插件，特别是配置分页插件时还有可能遇到很多稀奇古怪的，例如依赖版本冲突之类的问题。而使用 MyBatis-Plus 则完全没有这些顾虑，可以让用户专注于业务开发，它在数据库设计完成之后，能在极短时间内通过代码生成器将后台框架搭建起来，并且对于简单增删改查业务直接可以开始进行 Controller 层的接口编写，让多余、重复的操作和低级的文件配置、SQL 编写错误不在成为后台系统开发进度的阻碍。
