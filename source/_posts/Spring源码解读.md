---
title: Spring源码解读(-)
date: 2017-02-07 13:08:40
tags: 
    - spring
    - ioc
    - aop
    - 源码
---

### IOC
#### **设计理念**
先来看看接口设计预览图：
![IOC接口设计规范](http://jacobs.wanhb.cn/images/ioc1.jpg)


#### 初始化过程

容器的初始化过程是IOC实现的入口，过程如下：

> 1. Resource定位。BeanDefitioin的资源定位，由ResourceLoader通过统一的Resource接口完成
，这个Resource对各种形式的BeanDefinition使用都提供了统一接口。
2. BeanDefition的载入。这个载入过程是用户定义好的Bean表示成IOC容器内部的数据结构，而这个容器的数据
结构就是BeanDefition。BeanDefition实际上就是POJO对象在IOC容器中的抽象，通过这个BeanDefition定义
的数据结构，使得IOC容器能够方便地对POJO对象也就是Bean进行管理。
3. 向IOC容器注册这些BeanDefition的过程。调用BeanDefitionRegistry接口的实现来完成。把解析得到的BeanDefition向容器中进行注册。在IOC内部将BeanDefition注入到一个HasMap中去(BeanDefitionHolder),IOC容器就是通过这个HashMap来持有这些BeanDefition数据的。
4. **值得注意：**容器初始化过程不包括依赖注入的实现，Bean定义的载入和依赖注入是俩个独立的过程。依赖注入一般发生在第一次getBean的时候或者通过设置lazyinit实现预先注入Bean。 


AbstractApplicationContext定义了基本的refresh方法，其他的由子类去实现扩展


#### 注入过程
DefaultListableBeanFactory 是IOC容器的基础，FileSystemXmlApplicationContext、WebXmlApplicationContext都是建立在DefaultListableBeanFactory之上，实现自定义BeanDefinition的载入方式
IOC容器初始化的入口：Refresh方法。
IOC getBean实现原理：在使用DefaultListableBeanFactory 作为IOC容器的时候，它的基类是AbstractAutowireCapableBaenFacotoy。基类中判断是否实现了BeanFactoryAware接口，如果实现了，也就是实现了接口方法，这是将IOC容器设置到Bean自身定义的一个属性中去。
```
    if(bean instance BeanFactoryAware){
        ((BeanFactoryAware) bean).setBeanFactory(this);
    }
```
也即是如果需要回调容器一定需要实现BeanFactoryAware 接口

对于依赖注入，createBean根据Beandefinition生成了目标对象并且初始化了beanpostprocesser返回了代理对象。
populateBean则根据beandefinition的属性完成了bean的依赖注入。Autowire等之类的注入均在这个方法中完成。
getBean是依赖注入的起点，之后会调用createBean，在AbstractAutowireCapableBeanFactory中实现了这个方法，还对bean初始化进行了处理，实现了bean后置处理器等。注入过程如下：
![注入过程](http://jacobs.wanhb.cn/images/ioc3.jpg)


### AOP

#### AOP名词解释

> **方面（Aspect）**：
一个关注点的模块化，这个关注点实现可能另外横切多个对象。事务管理是J2EE应用中一个很好的横切关注点例子。方面用spring的 Advisor或拦截器实现。
**连接点（Joinpoint）:** 程序执行过程中明确的点，如方法的调用或特定的异常被抛出。
**通知（Advice）:** 在特定的连接点，AOP框架执行的动作。各种类型的通知包括“around”、“before”和“throws”通知。通知类型将在下面讨论。许多AOP框架包括Spring都是以拦截器做通知模型，维护一个“围绕”连接点的拦截器链。Spring中定义了四个advice: BeforeAdvice, AfterAdvice, ThrowAdvice和DynamicIntroductionAdvice
**切入点（Pointcut 一系列连接点的集合）:** 指定一个通知将被引发的一系列连接点的集合。AOP框架必须允许开发者指定切入点：例如，使用正则表达式。 Spring定义了Pointcut接口，用来组合MethodMatcher和ClassFilter，可以通过名字很清楚的理解， MethodMatcher是用来检查目标类的方法是否可以被应用此通知，而ClassFilter是用来检查Pointcut是否应该应用到目标类上
**引入（Introduction）:** 添加方法或字段到被通知的类。 Spring允许引入新的接口到任何被通知的对象。例如，你可以使用一个引入使任何对象实现 IsModified接口，来简化缓存。Spring中要使用Introduction, 可有通过DelegatingIntroductionInterceptor来实现通知，通过DefaultIntroductionAdvisor来配置Advice和代理类要实现的接口
**目标对象（Target Object）:** 包含连接点的对象。也被称作被通知或被代理对象。POJO
**AOP代理（AOP Proxy）:** AOP框架创建的对象，包含通知。 在Spring中，AOP代理可以是JDK动态代理或者CGLIB代理。
**织入（Weaving）:** 组装方面来创建一个被通知对象。这可以在编译时完成（例如使用AspectJ编译器），也可以在运行时完成。Spring和其他纯Java AOP框架一样，在运行时完成织入


#### **设计原理及流程**
Advice、PointCut、Advisor(通知器，组织起Advice与PointCut)

![接口设计](http://jacobs.wanhb.cn/images/aopinterface.jpg)
AopProxy代理对象生成过程：ProxyFactoryBean和ProxyFactory都提供了AOP的功能封装，但是ProxyFactoryBean与IOC进行了结合，利用BeanFactoryAware获取ApplicationContext,从而可以利用context对IOC注入的bean进行获取

初始化通知链—— >获取单例，没有的话去创建——>判断是否为接口，如果为接口使用JDK,如果不是使用CGlib最后返回AopProxy

#### **AOP调用**

invoke方法，里面逐个去应用配置好的拦截器链,在逐个应用之前先进行一系列的判断：
```
／／如果目标对象没有实现object的基本法方法：equals、如果目标对象没有实现object的基本方法： hashcode、根据代理对象的配置来调用服务
if(!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
  Boolean retVal3 = Boolean.valueOf(this.equals(args[0]));
  return retVal3;
}

if(!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
  Integer retVal2 = Integer.valueOf(this.hashCode());
  return retVal2;
}

if(method.getDeclaringClass() == DecoratingProxy.class) {
  Class retVal1 = AopProxyUtils.ultimateTargetClass(this.advised);
  return retVal1;
}

Object retVal;
if(!this.advised.opaque && method.getDeclaringClass().isInterface() && method.getDeclaringClass().isAssignableFrom(Advised.class)) {
  retVal = AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
  return retVal;
}

if(this.advised.exposeProxy) {
  oldProxy = AopContext.setCurrentProxy(proxy);
  setProxyContext = true;
}

target = targetSource.getTarget();
if(target != null) {
  targetClass = target.getClass();
}
```
然后获取配置好的拦截器：
```
List chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
```

判断是否为空，为空的话直接调用invokeJoinpointUsingReflection方法。这个方法直接调用目标方法的实现，代码如下：
```
public static Object invokeJoinpointUsingReflection(Object target, Method method, Object[] args) throws Throwable {
  try {
    ReflectionUtils.makeAccessible(method);
    return method.invoke(target, args);
  } catch (InvocationTargetException var4) {
    throw var4.getTargetException();
  } catch (IllegalArgumentException var5) {
    throw new AopInvocationException("AOP configuration seems to be invalid: tried calling method [" + method + "] on target [" + target + "]", var5);
  } catch (IllegalAccessException var6) {
    throw new AopInvocationException("Could not access method [" + method + "]", var6);
  }
}

如果不为空，则创建ReflectiveMethodInvocation传入chain（拦截器链）,然后调用proceed方法，这个方法去递归调用拦截器链中的invoke方法，代码如下（在拦截器的调用一节还会详细展开介绍）：
ReflectiveMethodInvocation invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
retVal = invocation.proceed();

public Object proceed() throws Throwable {
  if(this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
    return this.invokeJoinpoint();
  } else {
    Object interceptorOrInterceptionAdvice = this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
    if(interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
      InterceptorAndDynamicMethodMatcher dm = (InterceptorAndDynamicMethodMatcher)interceptorOrInterceptionAdvice;
      return dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)?dm.interceptor.invoke(this):this.proceed();
    } else {
      return ((MethodInterceptor)interceptorOrInterceptionAdvice).invoke(this);
    }
  }
}
```
对于最后的目标对象的调用：JDK 直接通过AopUtils的反射机制而cglib则是通过 MethodProxy完成调用，这是cglib自己封住好的功能。
```
retval=methodProxy.invoke(target,args);
```

#### **AOP拦截器链的调用**
了解了AOP的调用之后，再来看看AOP是怎么实现对目标对象增强的。
在运行拦截器链的拦截方法时，需要对代理方法完成一个匹配判断，通过这个匹配判断来决定是否满足切面增强的要求。确定是否执行拦截方法。
获取interceptors的操作是由advised的对象完成的。是一个AdvisedSupport对象。AdvisedSupport是ProxyFactoryBean的基类。在其中，我们可以看到getInterceptorsAndDynamicInterceptionAdvice 方法的实现：
```
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, Class<?> targetClass) {
  AdvisedSupport.MethodCacheKey cacheKey = new AdvisedSupport.MethodCacheKey(method);
  List cached = (List)this.methodCache.get(cacheKey);
  if(cached == null) {
    cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(this, method, targetClass);
    this.methodCache.put(cacheKey, cached);
  }

  return cached;
}
```
这里AdvisedSupport被配置成一个DefaultAdvisedSupport对象，里面实现了具体的getInterceptorsAndDynamicInterceptionAdvice方法。
```
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Advised config, Method method, Class<?> targetClass) {
  ArrayList interceptorList = new ArrayList(config.getAdvisors().length);
  Class actualClass = targetClass != null?targetClass:method.getDeclaringClass();
  boolean hasIntroductions = hasMatchingIntroductions(config, actualClass);
  AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
  Advisor[] var8 = config.getAdvisors();
  int var9 = var8.length;

  for(int var10 = 0; var10 < var9; ++var10) {
    Advisor advisor = var8[var10];
    MethodInterceptor[] interceptors1;
    if(advisor instanceof PointcutAdvisor) {
      PointcutAdvisor var20 = (PointcutAdvisor)advisor;
      if(config.isPreFiltered() || var20.getPointcut().getClassFilter().matches(actualClass)) {
        interceptors1 = registry.getInterceptors(advisor);
        MethodMatcher mm = var20.getPointcut().getMethodMatcher();
        if(MethodMatchers.matches(mm, method, actualClass, hasIntroductions)) {
          if(mm.isRuntime()) {
            MethodInterceptor[] var15 = interceptors1;
            int var16 = interceptors1.length;

            for(int var17 = 0; var17 < var16; ++var17) {
              MethodInterceptor interceptor = var15[var17];
              interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
            }
          } else {
            interceptorList.addAll(Arrays.asList(interceptors1));
          }
        }
      }
    } else if(advisor instanceof IntroductionAdvisor) {
      IntroductionAdvisor var19 = (IntroductionAdvisor)advisor;
      if(config.isPreFiltered() || var19.getClassFilter().matches(actualClass)) {
        interceptors1 = registry.getInterceptors(advisor);
        interceptorList.addAll(Arrays.asList(interceptors1));
      }
    } else {
      MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
      interceptorList.addAll(Arrays.asList(interceptors));
    }
  }

  return interceptorList;
}
```
DefaultAdvisorChainFactory 会通过一个AdvisorDapterRegister实现拦截器的注册，注册完成之后,List中的拦截器会被JDK生成的AopProxy中的代理对象的invoke调用，这里通过配置的Intercepters获得拦截器列表然后逐一应用在目标方法上。
如下：
```
public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
    ArrayList interceptors = new ArrayList(3);
    Advice advice = advisor.getAdvice();
    if(advice instanceof MethodInterceptor) {
      interceptors.add((MethodInterceptor)advice);
    }

    Iterator var4 = this.adapters.iterator();

    while(var4.hasNext()) {
      AdvisorAdapter adapter = (AdvisorAdapter)var4.next();
      if(adapter.supportsAdvice(advice)) {
        interceptors.add(adapter.getInterceptor(advisor));
      }
    }

    if(interceptors.isEmpty()) {
      throw new UnknownAdviceTypeException(advisor.getAdvice());
    } else {
      return (MethodInterceptor[])interceptors.toArray(new MethodInterceptor[interceptors.size()]);
    }
  }
```




