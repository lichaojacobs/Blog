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
![IOC接口设计规范](http://ol7zjjc80.bkt.clouddn.com/ioc1.jpg)


#### 资源定位与注册

容器的初始化过程是IOC实现的入口，过程如下：

> 1. Resource定位。BeanDefitioin的资源定位，由ResourceLoader通过统一的Resource接口完成
，这个Resource对各种形式的BeanDefinition使用都提供了统一接口。
2. BeanDefition的载入。这个载入过程是用户定义好的Bean表示成IOC容器内部的数据结构，而这个容器的数据
结构就是BeanDefition。BeanDefition实际上就是POJO对象在IOC容器中的抽象，通过这个BeanDefition定义
的数据结构，使得IOC容器能够方便地对POJO对象也就是Bean进行管理。
3. 向IOC容器注册这些BeanDefition的过程。调用BeanDefitionRegistry接口的实现来完成。把解析得到的BeanDefition向容器中进行注册。在IOC内部将BeanDefition注入到一个HasMap中去(BeanDefitionHolder),IOC容器就是通过这个HashMap来持有这些BeanDefition数据的。
4. **值得注意：**容器初始化过程不包括依赖注入的实现，Bean定义的载入和依赖注入是俩个独立的过程。依赖注入一般发生在第一次getBean的时候或者通过设置lazyinit实现预先注入Bean。 


1. AbstractApplicationContext定义了基本的refresh方法，其他的由子类去实现扩展,即AbstractApplicationContext是容器初始化的入口

2. DefaultListableBeanFactory是IOC容器的基础，FileSystemXmlApplicationContext、WebXmlApplicationContext都是建立在DefaultListableBeanFactory之上，实现自定义BeanDefinition的载入方式

#### 依赖注入
- 初始化过程完成的主要工作是在IOC容器中建立BeanDefinition数据映射。此过程中并没有实现IOC容器对Bean依赖关系进行注入。
- 对于依赖注入，其触发条件是用户第一次向IOC容器索要Bean时触发的。当然也可以通过控制lazy-init属性来让容器完成对bean的预实例化。
- **依赖注入的起点：** IOC容器接口BeanFactory中定义了一个getBean接口，这个接口的实现就是触发依赖注入的地方。可以在DefaultListableBeanFactory的类AbstractBeanFactory入手看看getBean的实现。
- SimpleInstsntiationStrategy类，这个Strategy是Spring用来生成Bean对象的默认类，提供了俩种实例化Java对象的方法，一种是通过BeanUtils，它使用了JVM的反射功能，一种是CGLIB来生成，代码如下:
```
@Override
	public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner) {
		// Don't override the class with CGLIB if no overrides.
		if (bd.getMethodOverrides().isEmpty()) {
			Constructor<?> constructorToUse;
			synchronized (bd.constructorArgumentLock) {
				constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
				if (constructorToUse == null) {
					final Class<?> clazz = bd.getBeanClass();
					if (clazz.isInterface()) {
						throw new BeanInstantiationException(clazz, "Specified class is an interface");
					}
					try {
						if (System.getSecurityManager() != null) {
							constructorToUse = AccessController.doPrivileged(new PrivilegedExceptionAction<Constructor<?>>() {
								@Override
								public Constructor<?> run() throws Exception {
									return clazz.getDeclaredConstructor((Class[]) null);
								}
							});
						}
						else {
							constructorToUse =	clazz.getDeclaredConstructor((Class[]) null);
						}
						bd.resolvedConstructorOrFactoryMethod = constructorToUse;
					}
					catch (Throwable ex) {
						throw new BeanInstantiationException(clazz, "No default constructor found", ex);
					}
				}
			}
			return BeanUtils.instantiateClass(constructorToUse);
		}
		else {
			// Must generate CGLIB subclass.
			return instantiateWithMethodInjection(bd, beanName, owner);
		}
	}
```
- **依赖注入最终步骤**：在实例化Bean对象生成的基础上，对于Bean对象生成以后，怎样对这些Bean对象的依赖关系处理好，完成整个依赖注入过程？即通过populateBean方法完成，这个方法在AbstractAutowireCapableBeanFactory中实现，AutoWire的依赖注入等，都做了集中处理。

- **Bean的初始化(InitializeBean)：**：在initializeBean方法中，需要使用Bean的名字，完成依赖注入以后的Bean对象，以及这个Bean对应的BeanDefinition。然后开始初始化工作：
> 
为类型是BeanNameAware的Bean设置Bean的名字
为类型是BeanClassLoaderAware的Bean设置类装载器，
类型是BeanFactoryAware的Bean设置自身所在的IOC容器以供回调使用，
对PostProcessBeforeInitialization/postAfterInitialization的回调和初始化属性init-method的处理等。
最后，就可以正常的使用由IOC容器托管的Bean了

- 
![注入过程](http://ol7zjjc80.bkt.clouddn.com/ioc3.jpg)

#### 讲讲预先注入
我们讲过依赖注入一般发生在用户第一次请求，但是也可以设置lazy-init属性实现预先依赖注入。这部分过程依然属于AbstractApplicationContext的 refresh方法中。在finishBeanFactoryInitialization的方法中，封装了lazy-init属性的处理，实际的处理是在DefaultListableBeanFactory这个基本容器的preInstantiateSingletons方法中完成的。该方法对单件Bean完成预先实例化。
```
 public void refresh() throws BeansException, IllegalStateException {
    Object var1 = this.startupShutdownMonitor;
    synchronized(this.startupShutdownMonitor) {
      this.prepareRefresh();
      ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
      this.prepareBeanFactory(beanFactory);

      try {
        this.postProcessBeanFactory(beanFactory);
        this.invokeBeanFactoryPostProcessors(beanFactory);
        this.registerBeanPostProcessors(beanFactory);
        this.initMessageSource();
        this.initApplicationEventMulticaster();
        this.onRefresh();
        this.registerListeners();
        //预先实例化入口
        this.finishBeanFactoryInitialization(beanFactory);
        this.finishRefresh();
      } catch (BeansException var9) {
        if(this.logger.isWarnEnabled()) {
          this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
        }
        this.destroyBeans();
        this.cancelRefresh(var9);
        throw var9;
      } finally {
        this.resetCommonCaches();
      }
    }
  }
  
  //在finishBeanFactoryInitialization方法中进行具体的处理过程
  protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    ...
    beanFactory.setTempClassLoader((ClassLoader)null);
    beanFactory.freezeConfiguration();
    beanFactory.preInstantiateSingletons();
  }
  
	public void preInstantiateSingletons() throws BeansException {
		List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
							@Override
							public Boolean run() {
								return ((SmartFactoryBean<?>) factory).isEagerInit();
							}
						}, getAccessControlContext());
					}
					else {
						isEagerInit = (factory instanceof SmartFactoryBean &&
								((SmartFactoryBean<?>) factory).isEagerInit());
					}
					if (isEagerInit) {
						getBean(beanName);
					}
				}
				else {
					getBean(beanName);
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged(new PrivilegedAction<Object>() {
						@Override
						public Object run() {
							smartSingleton.afterSingletonsInstantiated();
							return null;
						}
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
```

**存疑：** spring 2 中preInstantiateSingletons的实现是加了个Synchronized内置锁，而在当前版本中，这一步去掉了锁，why?

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

![接口设计](http://ol7zjjc80.bkt.clouddn.com/aopinterface.jpg)
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




