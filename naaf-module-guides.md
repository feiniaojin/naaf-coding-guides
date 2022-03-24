# NAAF项目结构及开发指南

> 本文档用于指导NAAF使用者将代码放置到适当的包中。

# 1. NAAF项目结构

详细项目结构，见https://github.com/feiniaojin/naaf-init

# 2.NAAF项目模块指南

## 2.1 naaf-commons

项目级别的公共包，用来放置通用的工具类。

【强制】**naaf-commons**包中只能引入外部公共类库，例如apache-commons。

【强制】**naaf-commons**包中不允许引入中间件客户端，例如Elastic Search、Redis的客户端，如果必须依赖中间件客户端，必须单独创建一个包，名字格式为：naaf-common-xxx，如naaf-common-redis。

【强制】**naaf-commons**中不允许有默认初始化的Bean。

【建议】一些类似POI处理的专业类库，建议也单独一个包进行放置。

【建议】不需要初始化、即可直接对外提供静态方法的工具类，建议命名为**XxxUtil**，例如**EncodeUtil**；需要初始化才能使用的工具类，建议命名为**XxxHelper**，例如**SpringContextHelper**。

## 2.2 naaf-domain
项目的数据库实体层。

【强制】本层数据实体由**naaf-generator**自动生成，不允许往生成的实体里加入自定义的注解或者逻辑，原因是字段变更会重新自动生成，替换时不注意的话会造成代码丢失。

【强制】项目中的数据库实体，生成时应扁平化，即不要抽象公共父类，与数据库字段一一对应。

## 2.3 naaf-adapter-dao

项目的DAO层，放置数据库访问对象。

【强制】基础Mapper类通过**naaf-generator**自动生成，不得在自动生成的Mapper类上加入自定义方法。如果需要，应在同一包下创建新的Mapper文件，名称格式为**XxxMapperEx**，代表对**XxxMapper**的扩展。

示例：

已经有了自动生成的**ItemMapper**，但是需要进行复杂查询

```java
public interface ItemMapper{
    //自动生成的数据操作方法
}
```

创建新的**ItemMapperEx**，里面定义需要的复杂查询方法

```java
public interface ItemMapperEx{
    /**
    * queryReq 查询条件
    */
    List<Item> list(ItemQueryReq queryReq)
}
```

【强制】数据库操作一律创建**Mapper**文件，显式书写SQL语句方便Code Review。

【强制】自定义的Mapper中，数据操作方法的入参在本层定义，例如ItemQueryReq，方法的返回值为基本类型或者naaf-model包中的数据库实体。

【强制】Mapper对应的XML文件，根据就近原则，一律放在该Mapper所在的module，路径为resources/mappers。

【强制】不允许在DAO层放置数据源等配置文件，数据源配置文件应当放置在**app**层。

## 2.4 naaf-adapter-rpc-consumer

本层用于处理外部RPC调用的接入逻辑。

在调用外部RPC接口时，通常会返回Response/Result对象，里面包含了返回码和返回数据。如果在业务代码中判断RPC调用的返回码是否正确，会造成Service代码臃肿不堪，因此，应当在本层处理外部RPC调用。

【强制】本层封装外部RPC调用，不允许向上层传递RPC调用的返回码，如调用异常，直接抛出自定义的运行时异常，NGR捕获后封装错误码返回。

【强制】RPC调用成功时，如果没有数据返回，直接向上层返回void；如果有数据返回，封装为本层的实体，不得直接返回RPC数据。

## 2.5 naaf-service

领域服务层，核心业务逻辑放置在本层。

【强制】本层向Controller层提供DTO包，用于接受请求和响应。Service层向Controller层提供的方法，其入参和返回值必定为基本类型或者DTO对象，不允许直接返回数据库实体。

示例：

```java
public interface ItemService{
    //ItemCmd、ItemView均为DTO对象
    ItemView getById(ItemCmd itemCmd);
}
```

【强制】本层与Controller层交互的DTO，命名规范为：数据变更请求的DTO，命名为**XXXCmd**，数据查询请求的DTO，命名为**XXXQuery**，查询结果返回的DTO，命名为**XXXView**。拒绝使用各种DO/VO/PO等。

【强制】本层使用DAO层提供的Mapper或者Repository访问数据库。Service层通过创建DAO层的数据请求对象，调用DAO层方法，得到数据库实体；在Service层向上返回时，需要将数据实体转成View类型的DTO对象，不得直接返回数据库实体。

【强制】本层引入validate组件，在作为Controller的请求入参的DTO中，添加校验规则进行参数校验，校验异常外抛，NGR统一处理。

示例：

```java
public class ItemCmd{
    @NotNull(message="id不能为空")
    private Long id;
}
```

【强制】本层定义业务异常，业务异常必须使用@ExceptionMapper进行注解，注解中定义异常码和提示信息。

【建议】同一个业务的异常，建议作为一组内部类。

示例：

```java
public class ItemExceptions{
    @ExceptionMapper(code=1024,msg="item不存在")
    public static class ItemNotFoundException extends RuntimeException{}
    
     @ExceptionMapper(code=2048,msg="item已存在")
    public static class ItemExistsException extends RuntimeException{}
}
```

## 2.6 naaf-ohs-web

ohs层之一，本层实现了web协议。本层引入SpringMVC相关包，在本层实现拦截器、Controller等逻辑。

【强制】Controller中的方法，不允许返回类似**ResponseBean**对象，应该直接返回业务结果，由NGR自动封装为**ResponseBean**。

示例：

```java
@RestController
@RequestMapping("/item")
public class ItemController {
    @Resource
    private ItemService itemService;

    @RequestMapping("/get")
    public ItemView get() {
        return itemService.get();
    }
}
```

> 示例中直接返回了ItemView这个DTO对象，不封装为ResponseBean。

【建议】通用、简单的参数校验，直接在方法入参DTO加@Validated注解进行，尽量不将校验逻辑引入到Controller代码中。

示例：

```java
@RestController
@RequestMapping("/item")
public class ItemController {
    @Resource
    private ItemService itemService;

    @RequestMapping("/idnull")
    public Item idnull(@Validated ItemQuery query) {
        return itemService.get();
    }
}

// ItemReq的代码如下
public class ItemQuery{
    @NotNull(message="id不能为空")
    private Long id;
}
```

## 2.7 naaf-ohs-rpc-provider

ohs层之一，用于对外暴露RPC服务。对外提供服务时，通常定义一系列RPC接口，本层实现这些接口。

## 2.8 naaf-ohs-schedule

ohs层之一，用于接入定时任务，依赖Service执行逻辑。

【强制】Schedule层依赖Service层执行逻辑，且不允许将定时任务客户端传入到Service层。

## 2.9 naaf-app

应用层，本层会将各个组件整合到一起，形成应用对外提供服务。在本层引入启动Starter类、Shell脚本、Dockerfile、以及各种配置文件。

【强制】工程启动的引导类、Shell脚本、Dockerfile、以及各种配置文件，统一放置在本层。