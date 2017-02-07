---
title: 记一次失败的debug经历
date: 2017-01-19 11:24:31
tags:
    - 成长
    - spring 
---

昨天使用spring aop的事务注解出现了业务中主动抛出异常却无法回滚脏数据的情形，然后崩溃的debug了一天，最后猛然发现不能通过this去掉加了增强的方法否则将拿不到代理对象。归根结底还是自己不甚理解事务实现的其中原理，才导致了bug的出现而不自知。

### 问题还原
如果对java如何实现底层的事务机制不太熟悉的话可以看看[java事务处理系列文章][1] 自己手动实现事务处理。

来看看我调用事务的错误示例代码：

```
@Component
class DemoService{

   public List<ExpressExportRow> notExpressedRecord() {
    ...
    changeOrderStatusToExpressing();
    ...
    return expressExportRowList;
  }
  

  @Transactional(value = "apolloTransactionManager",rollbackFor = Exception.class)
  public boolean changeOrderStatusToExpressing(Long orderId, Long orderGroupId) {
    boolean isSuccess = updateExpressRecordStatus(orderId, orderGroupId,
        OrderStatus.EXPRESSING.getValue());
    if (isSuccess) {
      isSuccess = apolloService.updateOrderStatus(orderId, orderGroupId,
          OrderStatus.EXPRESSING.getValue());
    }
    if (!isSuccess) {
      throw new RuntimeException("更改订单状态失败,订单号为: " + orderId);
    }

    return isSuccess;
  }
  }
```
首先DemoService通过@Component注解在容器加载的时候是确实注入进去了， 由于AOP 结合了IOC 的一部分功能（ProxyFactoryBean中实现）也就是在容器启动的过程中跟着在Demoservice封装成了aop代理对象保存在容器中。上面的代码错在直接在service 里面通过this引用去调增强的方法，结果导致方法不会应用相应的增强处理。来看看aop的处理流程图
![aop事务处理流程](http://jacobs.wanhb.cn/images/aop1.jpg)

### 解决办法
- 清楚了导出bug的原因和spring事务的基本原理之后，相应的解决办法就很简单，即我们只需要确保我们拿到的demoservice实例是通过容器注入进来的即可，于是可以以下解法：
```
@Component
class Resource{
@Resource
  DemoServcie demoService;
  
  public changeStatus(){
    demoService.changeOrderStatusToExpressing(...);
  }
 } 
```
通过@Resource注入进来的demoService必然式容器中的已经加载配置好的代理对象。这样就能成功应用事务增强。

- 或者也可以通过后置器BeanPostProcessor将自身注入进来，即在自身的service里完成调用
```
@Component  
public class DemoService implements BeanPostProcessor {  
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {  
        return bean;  
    }  
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {  
        if(bean instanceof BeanSelfAware) { 
            ((BeanSelfAware) bean).setSelf(bean);
        }  
        return bean;  
    }  
}  
```
但是这样会出现循环依赖问题。所以第一种方法最为合适。

### 总结
通过这次的低级bug再一次发现自己对源码不够深入，虽然前前后后看过了不少spring源码，但是也没有立即发现bug的存在，各方面理解还有待提高。继续努力吧

  [1]: http://www.cnblogs.com/davenkin/archive/2013/02/16/java-tranaction-1.html