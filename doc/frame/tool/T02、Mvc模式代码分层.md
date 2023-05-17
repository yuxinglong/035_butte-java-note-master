# 一、分层策略

MVC模式与代码分层策略，MVC全名是ModelViewController即模型－视图－控制器，作为一种软件设计典范，用一种业务逻辑、数据、界面显示分离的方法组织代码，将业务逻辑聚集到一个部件里面，在改进和个性化定制界面及用户交互的同时，不需要重新编写业务逻辑，这是一种开发模式，但并不是实际开发中代码的分层模式，通常SSM框架的后端代码分层如下：

![](https://images.gitee.com/uploads/images/2022/0212/203130_77817707_5064118.png "02-1.png")

- controller控制层：定义服务端接口，入参出参，和一些入参校验；
- service业务服务层：组装业务逻辑，业务校验，构建控制层需要的参数模型；
- dao数据交互层：提供服务层需要的数据查询方法，处理数据交互条件相关的逻辑；
- mapper持久层：基于mybatis框架需要的原生支持，目前很常用的持久层组件；

# 二、控制层

**1、Rest接口风格**

基于资源访问和处理的逻辑，使用不同风格的注解。例如资源新增，更新，查询，删除。

```java
/**
 * 新增
 */
@PostMapping("/insert")
public Integer insert (@RequestBody BaseInfo baseInfo){
    return baseInfoService.insert(baseInfo);
}
/**
 * 更新
 */
@PutMapping("/update/{id}")
public String update(@PathVariable(value = "id") Integer id,
                     @RequestBody BaseInfo baseInfo) {
    if (id<1){
        return "error";
    }
    baseInfo.setId(id);
    return "update="+baseInfoService.update(baseInfo);
}
/**
 * 主键查询
 */
@GetMapping("/detail/{id}")
public InfoModel detail(@PathVariable(value = "id") Integer id) {
    return baseInfoService.detail(id) ;
}
/**
 * 主键删除
 */
@DeleteMapping("/delete/{id}")
public String delete(@PathVariable(value = "id") Integer id) {
    baseInfoService.delete(id) ;
    return "SUS" ;
}
```

**2、接口复用度**

不建议接口高度复用，例如增删改查都各自对接接口即可，基本原则，不同的客户端端操作，对于独立的接口。

```java
/**
 * 列表加载
 */
@GetMapping("/list")
public List<BaseInfo> list() {
    return baseInfoService.list(new BaseInfoExample()) ;
}
/**
 * 列表搜索
 */
@PostMapping("/search")
public List<BaseInfo> search (@RequestParam("userName") String userName,
                              @RequestParam("phone") String phone) {
    return baseInfoService.search(userName,phone) ;
}
```

例如常见的list接口，list通常都有会按条件加载的search机制，而且搜索的判断条件很复杂，建议分为两个接口，从实际考虑，大部分场景下都是只使用list接口，很少使用search搜索。

**3、入参出参**

校验客户端必须条件，例如某某条件必填必选等，如果有问题，快速阻断请求链路，做到程序入口控制层拦截返回。

```java
@PutMapping("/update/{id}")
public String update(@PathVariable(value = "id") Integer id,
                     @RequestBody BaseInfo baseInfo) {
    if (id<1){
        return "error";
    }
    baseInfo.setId(id);
    return "update="+baseInfoService.update(baseInfo);
}
```

参数在三个以下，可以直接陈列入参，参数在三个或三个以上可以使用实体类统一封装。

```java
@PostMapping("/search")
public List<BaseInfo> search (@RequestParam("userName") String userName,
                              @RequestParam("phone") String phone) {
    return baseInfoService.search(userName,phone) ;
}
```

**4、参数处理**

出参格式处理度基本原则，服务器作为公共资源，避免非必要操作，例如客户端可自行判断返回值是否为空，null等，或者一些常见格式处理，利用客户端适当分担服务器压力。

# 三、业务服务层

**1、业务校验**

例如传入订单号，经过数据库层查询，没有订单数据，这里称为业务性质的异常，代码本身没有问题，但是业务逻辑无法正常执行。

```java
public InfoModel detail(Integer id){
    BaseInfo baseInfo = baseInfoDao.selectByPrimaryKey(id) ;
    if (baseInfo != null){
        DetailInfoEntity detailInfoEntity = detailInfoDao.getById(id);
        if (detailInfoEntity == null){
            LOG.info("id="+id+"数据缺失 DetailInfo");
        }
        return buildModel(baseInfo,detailInfoEntity) ;
    }
    LOG.info("id="+id+"数据完全缺失");
    return null ;
}
```

**2、组装业务逻辑**

通常情况下服务层作为逻辑做复杂的一块，用来拼接业务核心步骤，可以通过业务逻辑判定，一步一步执行程序，避免在程序入口做大量可能用到的对象创建和需求数据查询。

```java
public int insert (BaseInfo record){
    record.setCreateTime(new Date());
    int insertFlag = baseInfoDao.insert(record);
    if (insertFlag > 0){
        DetailInfoEntity detailInfoEntity = new DetailInfoEntity();
        detailInfoEntity.setUserId(record.getId());
        detailInfoEntity.setCreateTime(record.getCreateTime());
        if(detailInfoDao.save(detailInfoEntity)){
            return insertFlag ;
        }
    }
    return insertFlag;
}
```

**3、数据模型构建**

通常情况业务层是偏复杂的，如果想关快速理解业务层，可以对复杂的业务方法，在提供一个返参构建的方法，用来处理服务层要向控制层回传的参数，这样可以让重度的服务层方法变的清晰。

```java
private InfoModel buildModel (BaseInfo baseInfo,DetailInfoEntity detailInfo){
    InfoModel infoModel = new InfoModel() ;
    infoModel.setBaseInfo(baseInfo);
    infoModel.setDetailInfoEntity(detailInfo);
    return infoModel ;
}
```

# 四、数据交互层

**1、逆向工程**

这里以使用mybatis框架或者mybatis-plus框架作为参考。如果是mybatis框架，建议逆向工程的模板代码不做自定义的修改，如果需要自定义方法，在mapper和xml层面再自定义一个扩展文件，用来存放自定义的方法和SQL逻辑，这样避免表结构变动大引发的强烈不适。

![](https://images.gitee.com/uploads/images/2022/0212/203144_d7c70671_5064118.png "02-2.png")

当然现在大部分都会mybatis-plus作为持久层组件，可以避免上述问题。

**2、数据交互**

针对业务层的需要，提供相应的数据查询方法，只处理与数据库交互的逻辑，避免出现业务逻辑，尤其在分布式架构下，不同服务的数据查询和组装，不应该出现在该层。

```java
public interface BaseInfoDao {

    int insert(BaseInfo record);

    List<BaseInfo> selectByExample(BaseInfoExample example);

    int updateByPrimaryKey(BaseInfo record);

    BaseInfo selectByPrimaryKey(Integer id);

    int deleteByPrimaryKey(Integer id);

    BaseInfo getById (Integer id) ;
}
```

**源码参考：** https://gitee.com/cicadasmile/data-manage-parent/tree/master/heap04-format-tool/case02-mvc-style