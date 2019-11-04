---
title: Spring源码解析--事务处理
date: 2017-02-12 00:42:30
tags: 
    - spring
    - ioc
    - aop
    - 源码
    - 事务处理
    
---

### 事务处理相关类的层次结构

![事务处理相关类的层次结构](http://jacobs.wanhb.cn/images/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-02-11%20%E4%B8%8B%E5%8D%887.54.25.png)

在 Spring事务处理中，可以通过设计一个TransactionProxyFactoryBean来使用AOP功能，通过它可以生成Proxy代理对象。在代理对象中，通过TranscationInterceptor来完成对代理对象方法的拦截。实现声明式事务处理时，是AOP和IOC集成的部分，而对于具体的事物处理实现，是通过设计PlatformTransactionManager，AbstractPlatforTransactionmanager以及一系列具体事务处理器来实现的。PlatformTransactionManager又实现了TransactionInterceptor，这样就能将一系列处理给串联起来。

### Spring声明式事务处理

#### 设计原理与过程
在实现声明式的事务处理时，常用的方式是结合IOC容器和Spring已有的TransactionProxyFactoryBean对事务管理进行配置，实现可分为以下几个步骤：

- 读取和处理在IOC容器中配置的事务处理属性，并转化为Spring事务处理需要的内部数据结构。
- Spring事务处理模块实现统一的事务处理过程。
- 底层的事务处理实现。Spring委托给具体的事务处理器来完成。

![建立事务处理对象时序图](http://jacobs.wanhb.cn/images/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-02-11%20%E4%B8%8B%E5%8D%888.23.09.png)

从TransactionProxyFactoryBean入手，通过代码来了解Spring是如何通过AOP功能来完成事务管理配置的，从图中可以看到Spring为声明式事务处理的实现所做的一些准备工作：包括为AOP配置基础设施，这些基础设施包括设置拦截器TransactionInterceptor、通过DefaultPointcutAdvisor或TransactionAttributeSourceAdvisor。同时，在TransactionProxyFactoryBean的实现中，还可以看到注入进来的PlatformTransactionManager和事务处理属性TransactionAttribute等。

```
public class TransactionProxyFactoryBean extends AbstractSingletonProxyFactoryBean implements BeanFactoryAware {
  private final TransactionInterceptor transactionInterceptor = new TransactionInterceptor();／／这个拦截器通过AOP发挥作用，通过这个拦截器的实现，Spring封装了事务处理实现
  private Pointcut pointcut;

  public TransactionProxyFactoryBean() {
  }

  public void setTransactionManager(PlatformTransactionManager transactionManager) {
    this.transactionInterceptor.setTransactionManager(transactionManager);
  }
  //通过依赖注入的事务属性以Properties的形式出现，把BeanDefinition中读到的事务管理的属性信息注入到TransactionInterceptor中
  public void setTransactionAttributes(Properties transactionAttributes) {
    this.transactionInterceptor.setTransactionAttributes(transactionAttributes);
  }

  public void setTransactionAttributeSource(TransactionAttributeSource transactionAttributeSource) {
    this.transactionInterceptor.setTransactionAttributeSource(transactionAttributeSource);
  }

  public void setPointcut(Pointcut pointcut) {
    this.pointcut = pointcut;
  }

  public void setBeanFactory(BeanFactory beanFactory) {
    this.transactionInterceptor.setBeanFactory(beanFactory);
  }
//这里创建Spring  AOP对事务处理的Advisor
  protected Object createMainInterceptor() {
    this.transactionInterceptor.afterPropertiesSet();//事务处理完成AOP配置的地方
    return this.pointcut != null?new DefaultPointcutAdvisor(this.pointcut, this.transactionInterceptor):new TransactionAttributeSourceAdvisor(this.transactionInterceptor);
  }

  protected void postProcessProxyFactory(ProxyFactory proxyFactory) {
    proxyFactory.addInterface(TransactionalProxy.class);
  }
}

```

完成了AOP配置，Spring的TransactionInterceptor配置是IOC容器完成Bean的依赖注入时，通过initializeBean方法被调用。

   在建立TransactionProxyFactoryBean的事务处理拦截器的时候， afterPropertiesSet方法首先对 ProxyFactoryBean的目标Bean设置进行检查，如果这个目标Bean的设置是正确的，就会创建ProxyFactory对象，从而实现AOP的使用。

```
public void afterPropertiesSet() {
  if(this.getTransactionManager() == null && this.beanFactory == null) {
    throw new IllegalStateException("Set the \'transactionManager\' property or make sure to run within a BeanFactory containing a PlatformTransactionManager bean!");
  } else if(this.getTransactionAttributeSource() == null) {
    throw new IllegalStateException("Either \'transactionAttributeSource\' or \'transactionAttributes\' is required: If there are no transactional methods, then don\'t use a transaction aspect.");
  }
}
```

#### 事务处理配置的读入
在AOP配置完成的基础上，以TransactionAttributeSourceAdvisor的实现为入口，了解具体的事务属性配置是如何读入的，实现如下：
```
private TransactionInterceptor transactionInterceptor;／／同样需要AOP中用到的Interceptro和Pointcut，通过内部类，调用TransactionInterceptor来得到事务的配置属性，在对Proxy的方法进行匹配调用时，会使用到这些配置属性。
private final TransactionAttributeSourcePointcut pointcut = new TransactionAttributeSourcePointcut() {
  protected TransactionAttributeSource getTransactionAttributeSource() {
    return TransactionAttributeSourceAdvisor.this.transactionInterceptor != null?TransactionAttributeSourceAdvisor.this.transactionInterceptor.getTransactionAttributeSource():null;
  }
};
```
在声明式事务处理中，通过对目标对象的方法调用进行拦截实现，这个拦截通过AOP发挥作用。在AOP中，对于拦截的启动，首先需要对方法调用是否需要拦截进行判断，依据时那些在TransactionProxyFactoryBean中为目标对象设置的事务属性。这个匹配判断在TransactionAttributeSourcePointcut中完成。实现如下：

```
public boolean matches(Method method, Class<?> targetClass) {
  if(TransactionalProxy.class.isAssignableFrom(targetClass)) {
    return false;
  } else {
    TransactionAttributeSource tas = this.getTransactionAttributeSource();
    return tas == null || tas.getTransactionAttribute(method, targetClass) != null;
  }
}
```
在方法中，首先把事务方法的属性配置读取到TransactionAttributeSource对象中，有了这些事务处理的配置以后，根据当前方法调用的method对象和目标对象，对是否需要启动事务处理拦截器进行判断。

在Pointcut的matches判断过程中，会用到transactionAttributeSource对象，这个transactionAttributeSource对象是在对TransactionInterceptor进行依赖注入时就配置好的，它的设置是在TransactionInterceptor的基类TransactionAspectSupport中完成的。配置的是一个NameMatchTransactionAttributeSouce对象。

```
public void setTransactionAttributes(Properties transactionAttributes) {
  NameMatchTransactionAttributeSource tas = new NameMatchTransactionAttributeSource();
  tas.setProperties(transactionAttributes);
  this.transactionAttributeSource = tas;
}
```

可知，NameMatchTransactionAttributeSouce作为TransacionAttributeSource的具体实现，是实际完成事务处理属性读入和匹配的地方。对于NameMatchTransactionAttributeSouce是怎样实现事务处理属性的读入和匹配的，可看如下代码：

```
public void setProperties(Properties transactionAttributes) {//设置配置的事务方法
  TransactionAttributeEditor tae = new TransactionAttributeEditor();
  Enumeration propNames = transactionAttributes.propertyNames();

  while(propNames.hasMoreElements()) {
    String methodName = (String)propNames.nextElement();
    String value = transactionAttributes.getProperty(methodName);
    tae.setAsText(value);
    TransactionAttribute attr = (TransactionAttribute)tae.getValue();
    this.addTransactionalMethod(methodName, attr);
  }

}
private Map<String, TransactionAttribute> nameMap = new HashMap();


public void addTransactionalMethod(String methodName, TransactionAttribute attr) {
  if(logger.isDebugEnabled()) {
    logger.debug("Adding transactional method [" + methodName + "] with attribute [" + attr + "]");
  }

  this.nameMap.put(methodName, attr);
}
／／对调用的方法进行判断，判断它是否是事务方法，如果是，那么取出相应的事务配置属性
public TransactionAttribute getTransactionAttribute(Method method, Class<?> targetClass) {
  if(!ClassUtils.isUserLevelMethod(method)) {
    return null;
  } else {
    String methodName = method.getName();／／判断当前目标调用的方法与配置的事务方法是否直接匹配
    TransactionAttribute attr = (TransactionAttribute)this.nameMap.get(methodName);
    if(attr == null) {//如果不能直接匹配，就通过调用PatternMatchUtils的simpleMatch方法来进行匹配判断。
      String bestNameMatch = null;
      Iterator var6 = this.nameMap.keySet().iterator();

      while(true) {
        String mappedName;
        do {
          do {
            if(!var6.hasNext()) {
              return attr;
            }

            mappedName = (String)var6.next();
          } while(!this.isMatch(methodName, mappedName));
        } while(bestNameMatch != null && bestNameMatch.length() > mappedName.length());

        attr = (TransactionAttribute)this.nameMap.get(mappedName);
        bestNameMatch = mappedName;
      }
    } else {
      return attr;
    }
  }
}
／／事务方法的匹配判断，详细的匹配过程在PatternMatchUtils中实现
protected boolean isMatch(String methodName, String mappedName) {
  return PatternMatchUtils.simpleMatch(mappedName, methodName);
}
```

#### 事务处理拦截器的设计与实现：

经过TransactionProxyFactoryBean的AOP包装，此时如果对目标对象进行方法调用，起作用的对象实际傻姑娘是一个Proxy代理对象。对目标对象方法的调用，不会直接作用在TransactionProxyFactoryBean设置的目标对象上。而是会被设置的事务处理器拦截。而在TransactionProxyFactoryBean的AOP实现中，获取Proxy对象的过程并不复杂，TransactionProxyFactoryBean作为一个FactoryBean，对Bean对象的引用通过getObejct方法来得到的：

```
public Object getObject() { ／／TransactionProxyFactoryBean 的父类 AbstractSingletonProxyFactoryBean中
／／返回的是一个Proxy，是ProxyFactory生成的AOP代理，已经封装了对事务处理的拦截器设置
  if(this.proxy == null) {
    throw new FactoryBeanNotInitializedException();
  } else {
    return this.proxy;
  }
}
```
对于AOP代理对象的作用方法入口，我们一般都知道invoke方法，这个invke方法在事务处理拦截器TransactionInterceptor中，实现如下：

```
public Object invoke(final MethodInvocation invocation) throws Throwable {
  Class targetClass = invocation.getThis() != null?AopUtils.getTargetClass(invocation.getThis()):null;
  return this.invokeWithinTransaction(invocation.getMethod(), targetClass, new InvocationCallback() {
    public Object proceedWithInvocation() throws Throwable {
      return invocation.proceed();
    }
  });
}
protected Object invokeWithinTransaction(Method method, Class<?> targetClass, final TransactionAspectSupport.InvocationCallback invocation) throws Throwable {
  final TransactionAttribute txAttr = this.getTransactionAttributeSource().getTransactionAttribute(method, targetClass);／／这里读取事务的属性配置，通过TransactionAttributeSource对象取得
  final PlatformTransactionManager tm = this.determineTransactionManager(txAttr);／／根据TransactionProxyFactoryBean的配置信息获得具体的事务处理器
  final String joinpointIdentification = this.methodIdentification(method, targetClass, txAttr);
  if(txAttr != null && tm instanceof CallbackPreferringPlatformTransactionManager) {
    try {
      Object ex1 = ((CallbackPreferringPlatformTransactionManager)tm).execute(txAttr, new TransactionCallback() {
        public Object doInTransaction(TransactionStatus status) {
          TransactionAspectSupport.TransactionInfo txInfo = TransactionAspectSupport.this.prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);／／创建事务，同时把事务过程中得到的信息放到TransactionInfo中去，TransactionInfo是保存当前事务状态的对象。

          TransactionAspectSupport.ThrowableHolder var4;
          try {
            Object ex = invocation.proceedWithInvocation();
            return ex;
          } catch (Throwable var8) {
            if(txAttr.rollbackOn(var8)) {
              if(var8 instanceof RuntimeException) {
                throw (RuntimeException)var8;
              }

              throw new TransactionAspectSupport.ThrowableHolderException(var8);
            }

            var4 = new TransactionAspectSupport.ThrowableHolder(var8);
          } finally {
            TransactionAspectSupport.this.cleanupTransactionInfo(txInfo);
          }

          return var4;
        }
      });
      if(ex1 instanceof TransactionAspectSupport.ThrowableHolder) {
        throw ((TransactionAspectSupport.ThrowableHolder)ex1).getThrowable();
      } else {
        return ex1;
      }
    } catch (TransactionAspectSupport.ThrowableHolderException var14) {
      throw var14.getCause();
    }
  } else {
    TransactionAspectSupport.TransactionInfo ex = this.createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
    Object retVal = null;

    try {
      retVal = invocation.proceedWithInvocation();／／这里的调用使处理沿着拦截器链进行，使最后目标对象的方法得到调用
    } catch (Throwable var15) {
      this.completeTransactionAfterThrowing(ex, var15);／／如果事务处理方法中调用出现了异常，事务处理如何进行需要根据具体情况考虑是否会滚或者提交
      throw var15;
    } finally {
      this.cleanupTransactionInfo(ex);
    }

    this.commitTransactionAfterReturning(ex);//这里通过事务处理器来对事务进行提交
    return retVal;
  }
}
```

对于Spring而言，事务管理实际上是通过一个TransactionInfo对象来完成的，在该对象中，封装了事务对象和事务处理的状态信息，这是事务处理的抽象。在这一步完成以后，会对拦截器链进行处理，因为有可能在该事务对象中还配置了除事务处理AOP之外的其他拦截器，在结束对拦截器链处理之后，会对 TransactionInfo中的信息进行更新，以反映最近的事务处理情况，在这个时候，也就完成了事务提交的准备，通过调用事务处理器PlatformTransactionManager的commitTransactionAfterReturning方法来完成事务的提交。这个提交的处理过程已经封装在事务处理器中了，而与具体数据源相关的处理过程，最终委托给相关的事务处理器完成，如：DataSourceTransactionManager、HibernateTransactionManager等。

![事务提交时序图](http://jacobs.wanhb.cn/images/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-02-11%20%E4%B8%8B%E5%8D%8810.51.15.png)

这个invoke方法的实现中，可以看到整个事务处理在AOP拦截器中实现的全过程。同时，它也是Spring采用AOP封装事务处理和实现声明式事务处理的核心部分。

### Spring事务处理的设计与实现

#### Spring事务传播属性

> PROPAGATION_REQUIRED -- 支持当前事务，如果当前没有事务，就新建一个事务。这是最常见的选择。 
> PROPAGATION_SUPPORTS -- 支持当前事务，如果当前没有事务，就以非事务方式执行。 
> PROPAGATION_MANDATORY -- 支持当前事务，如果当前没有事务，就抛出异常。 
> PROPAGATION_REQUIRES_NEW -- 新建事务，如果当前存在事务，把当前事务挂起。 
> PROPAGATION_NOT_SUPPORTED -- 以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。 
> PROPAGATION_NEVER -- 以非事务方式执行，如果当前存在事务，则抛出异常。  PROPAGATION_NESTED --
> 如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则进行与PROPAGATION_REQUIRED类似的操作。 
> 前六个策略类似于EJB CMT，第七个（PROPAGATION_NESTED）是Spring所提供的一个特殊变量。

#### 事务的创建

声明式事务中，TransactionInterceptor拦截器的invoke方法作为事务处理实现的起点，invoke方法中createTransactionIfNeccessary方法作为事务创建的入口。以下是createTransactionIfNeccessary方法的时序图

![createTransactionIfNeccessary方法的时序图](http://jacobs.wanhb.cn/images/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-02-11%20%E4%B8%8B%E5%8D%8811.09.41.png)

在createTransactionIfNeccessary中首先会向AbstractTransactionManager执行getTransaction，这个获取Transaction事务对象的过程，在AbstractTransactionManager中需要对事务不同的情况作出处理，然后创建一个TransactionStatus，并把这个TransactionStatus设置到对应的TransactionInfo中去，同时将TransactionInfo和当前的线程绑定，从而完成事务的创建过程。TransactionStatus和TransactionInfo这俩个对象持有的数据是事务处理器对事务进行处理的主要依据。对这俩个对象的使用贯穿整个事务处理的全过程。

```
protected TransactionAspectSupport.TransactionInfo createTransactionIfNecessary(PlatformTransactionManager tm, final TransactionAttribute txAttr, final String joinpointIdentification) {
  if(txAttr != null && ((TransactionAttribute)txAttr).getName() == null) {
    txAttr = new DelegatingTransactionAttribute((TransactionAttribute)txAttr) {
      public String getName() {
        return joinpointIdentification;
      }
    };
  }

  TransactionStatus status = null;
  if(txAttr != null) {
    if(tm != null) {
      status = tm.getTransaction((TransactionDefinition)txAttr);／／这里使用了定义好的事务方法的配置信息。事务创建由事务处理器来完成，同时返回TransactionStatus来记录当前的事务状态，包括已经创建的事务。
    } else if(this.logger.isDebugEnabled()) {
      this.logger.debug("Skipping transactional joinpoint [" + joinpointIdentification + "] because no transaction manager has been configured");
    }
  }

  return this.prepareTransactionInfo(tm, (TransactionAttribute)txAttr, joinpointIdentification, status);
}

protected TransactionAspectSupport.TransactionInfo prepareTransactionInfo(PlatformTransactionManager tm, TransactionAttribute txAttr, String joinpointIdentification, TransactionStatus status) {
  TransactionAspectSupport.TransactionInfo txInfo = new TransactionAspectSupport.TransactionInfo(tm, txAttr, joinpointIdentification);
  if(txAttr != null) {
    if(this.logger.isTraceEnabled()) {
      this.logger.trace("Getting transaction for [" + txInfo.getJoinpointIdentification() + "]");
    }

    txInfo.newTransactionStatus(status);
  } else if(this.logger.isTraceEnabled()) {
    this.logger.trace("Don\'t need to create transaction for [" + joinpointIdentification + "]: This method isn\'t transactional.");
  }

  txInfo.bindToThread();
  return txInfo;
}
```
getTansaction实现如下：

```
@Override
public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
       Object transaction = doGetTransaction();

       // Cache debug flag to avoid repeated checks.
       boolean debugEnabled = logger.isDebugEnabled();

       if (definition == null) {
              // Use defaults if no transaction definition given.
              definition = new DefaultTransactionDefinition();
       }

       if (isExistingTransaction(transaction)) {
              // Existing transaction found -> check propagation behavior to find out how to behave.
              return handleExistingTransaction(definition, transaction, debugEnabled);
       }

       // Check definition settings for new transaction.
       if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
              throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
       }
／／没有事务存在，需要根据事务传播属性设置来创建事务，这里会看到事务传播属性的设置：mandatory、required required_new nested等
       // No existing transaction found -> check propagation behavior to find out how to proceed.
       if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
              throw new IllegalTransactionStateException(
                            "No existing transaction found for transaction marked with propagation 'mandatory'");
       }
       else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
                     definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
                     definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
              SuspendedResourcesHolder suspendedResources = suspend(null);
              if (debugEnabled) {
                     logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
              }
              try {
                     boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
                     DefaultTransactionStatus status = newTransactionStatus(
                                   definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
                     doBegin(transaction, definition);
                     prepareSynchronization(status, definition);
                     return status;
              }
              catch (RuntimeException ex) {
                     resume(null, suspendedResources);
                     throw ex;
              }
              catch (Error err) {
                     resume(null, suspendedResources);
                     throw err;
              }
       }
       else {
              // Create "empty" transaction: no actual transaction, but potentially synchronization.
              if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
                     logger.warn("Custom isolation level specified but no actual transaction initiated; " +
                                   "isolation level will effectively be ignored: " + definition);
              }
              boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
              return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
       }
}
```

handleExsitingTransaction方法是理解Spring事务传播属性的关键：

```
/**
* Create a TransactionStatus for an existing transaction.
*/
private TransactionStatus handleExistingTransaction(
              TransactionDefinition definition, Object transaction, boolean debugEnabled)
              throws TransactionException {

       if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
              throw new IllegalTransactionStateException(
                            "Existing transaction found for transaction marked with propagation 'never'");
       }

       if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
              if (debugEnabled) {
                     logger.debug("Suspending current transaction");
              }
              Object suspendedResources = suspend(transaction);
              boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
              return prepareTransactionStatus(
                            definition, null, false, newSynchronization, debugEnabled, suspendedResources);
       }

       if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
              if (debugEnabled) {
                     logger.debug("Suspending current transaction, creating new transaction with name [" +
                                   definition.getName() + "]");
              }
              SuspendedResourcesHolder suspendedResources = suspend(transaction);
              try {
                     boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
                     DefaultTransactionStatus status = newTransactionStatus(
                                   definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
                     doBegin(transaction, definition);
                     prepareSynchronization(status, definition);
                     return status;
              }
              catch (RuntimeException beginEx) {
                     resumeAfterBeginException(transaction, suspendedResources, beginEx);
                     throw beginEx;
              }
              catch (Error beginErr) {
                     resumeAfterBeginException(transaction, suspendedResources, beginErr);
                     throw beginErr;
              }
       }

       if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
              if (!isNestedTransactionAllowed()) {
                     throw new NestedTransactionNotSupportedException(
                                   "Transaction manager does not allow nested transactions by default - " +
                                   "specify 'nestedTransactionAllowed' property with value 'true'");
              }
              if (debugEnabled) {
                     logger.debug("Creating nested transaction with name [" + definition.getName() + "]");
              }
              if (useSavepointForNestedTransaction()) {／／在Spring管理的事务中，创建事务保存点
                     // Create savepoint within existing Spring-managed transaction,
                     // through the SavepointManager API implemented by TransactionStatus.
                     // Usually uses JDBC 3.0 savepoints. Never activates Spring synchronization.
                     DefaultTransactionStatus status =
                                   prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
                     status.createAndHoldSavepoint();
                     return status;
              }
              else {
                     // Nested transaction through nested begin and commit/rollback calls.
                     // Usually only for JTA: Spring synchronization might get activated here
                     // in case of a pre-existing JTA transaction.
                     boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
                     DefaultTransactionStatus status = newTransactionStatus(
                                   definition, transaction, true, newSynchronization, debugEnabled, null);
                     doBegin(transaction, definition);
                     prepareSynchronization(status, definition);
                     return status;
              }
       }

       // Assumably PROPAGATION_SUPPORTS or PROPAGATION_REQUIRED.
       if (debugEnabled) {
              logger.debug("Participating in existing transaction");
       }
       if (isValidateExistingTransaction()) {
              if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
                     Integer currentIsolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
                     if (currentIsolationLevel == null || currentIsolationLevel != definition.getIsolationLevel()) {
                            Constants isoConstants = DefaultTransactionDefinition.constants;
                            throw new IllegalTransactionStateException("Participating transaction with definition [" +
                                          definition + "] specifies isolation level which is incompatible with existing transaction: " +
                                          (currentIsolationLevel != null ?
                                                        isoConstants.toCode(currentIsolationLevel, DefaultTransactionDefinition.PREFIX_ISOLATION) :
                                                        "(unknown)"));
                     }
              }
              if (!definition.isReadOnly()) {
                     if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
                            throw new IllegalTransactionStateException("Participating transaction with definition [" +
                                          definition + "] is not marked as read-only but existing transaction is");
                     }
              }
       }
       boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
       return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
}
```

#### 事务挂起

```
protected final SuspendedResourcesHolder suspend(Object transaction) throws TransactionException {／／返回的SuspendedResourcesHolder会作为参数传给TransactionStatus
       if (TransactionSynchronizationManager.isSynchronizationActive()) {
              List<TransactionSynchronization> suspendedSynchronizations = doSuspendSynchronization();
              try {
                     Object suspendedResources = null;／／把挂起事务的处理交给具体事务处理器去完成，如果具体的事务处理器不支持事务挂起，则默认抛出TransactionSuspensionNotSupportedException
                     if (transaction != null) {
                            suspendedResources = doSuspend(transaction);
                     }//这里在线程中保存与事务处理有关的信息，并重置线程中相关的ThreadLocal变量
                     String name = TransactionSynchronizationManager.getCurrentTransactionName();
                     TransactionSynchronizationManager.setCurrentTransactionName(null);
                     boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
                     TransactionSynchronizationManager.setCurrentTransactionReadOnly(false);
                     Integer isolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
                     TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(null);
                     boolean wasActive = TransactionSynchronizationManager.isActualTransactionActive();
                     TransactionSynchronizationManager.setActualTransactionActive(false);
                     return new SuspendedResourcesHolder(
                                   suspendedResources, suspendedSynchronizations, name, readOnly, isolationLevel, wasActive);
              }
              catch (RuntimeException ex) {
                     // doSuspend failed - original transaction is still active… 如果处理失败，则恢复原始的事务
                     doResumeSynchronization(suspendedSynchronizations);
                     throw ex;
              }
              catch (Error err) {
                     // doSuspend failed - original transaction is still active...
                     doResumeSynchronization(suspendedSynchronizations);
                     throw err;
              }
       }
       else if (transaction != null) {
              // Transaction active but no synchronization active.
              Object suspendedResources = doSuspend(transaction);
              return new SuspendedResourcesHolder(suspendedResources);
       }
       else {
              // Neither transaction nor synchronization active.
              return null;
       }
}

/**
 * Reactivate transaction synchronization for the current thread
 * and resume all given synchronizations.
 * @param suspendedSynchronizations List of TransactionSynchronization objects
 */doSuspend 失败则恢复事务
private void doResumeSynchronization(List<TransactionSynchronization> suspendedSynchronizations) {
       TransactionSynchronizationManager.initSynchronization();／／维护着ThreadLocal变量
       for (TransactionSynchronization synchronization : suspendedSynchronizations) {
              synchronization.resume();
              TransactionSynchronizationManager.registerSynchronization(synchronization);
       }
}

```

#### 事务的提交
在声明式事务处理中，事务的提交在TransactionInteceptor的invoke方法中实现：
```
commitTransactionAfterReturning(txInfo)
```
txInfo是TransactionInfo对象，是创建事务时生成的。同时，Spring的事务管理框架的生成的TransactionStatus对象就包含在TransactionInfo对象中。commitTransactionAfterReturning具体实现如下：

```
protected void commitTransactionAfterReturning(TransactionAspectSupport.TransactionInfo txInfo) {
  if(txInfo != null && txInfo.hasTransaction()) {
    if(this.logger.isTraceEnabled()) {
      this.logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() + "]");
    }

    txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
  }

}
```

调用具体的事务管理器实现。而在事务管理器中的实现在AbstractPlatformTransactionManager中存在一个模版：
```
/**
* This implementation of commit handles participating in existing
* transactions and programmatic rollback requests.
* Delegates to {@code isRollbackOnly}, {@code doCommit}
* and {@code rollback}.
* @see org.springframework.transaction.TransactionStatus#isRollbackOnly()
* @see #doCommit
* @see #rollback
*/
@Override
public final void commit(TransactionStatus status) throws TransactionException {
       if (status.isCompleted()) {
              throw new IllegalTransactionStateException(
                            "Transaction is already completed - do not call commit or rollback more than once per transaction");
       }

       DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
       if (defStatus.isLocalRollbackOnly()) {／／如果事务处理过程中发生了异常，调用回滚。
              if (defStatus.isDebug()) {
                     logger.debug("Transactional code has requested rollback");
              }
              processRollback(defStatus);
              return;
       }
       if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
              if (defStatus.isDebug()) {
                     logger.debug("Global transaction is marked as rollback-only but transactional code requested commit");
              }／／处理回滚
              processRollback(defStatus);
              // Throw UnexpectedRollbackException only at outermost transaction boundary
              // or if explicitly asked to.
              if (status.isNewTransaction() || isFailEarlyOnGlobalRollbackOnly()) {
                     throw new UnexpectedRollbackException(
                                   "Transaction rolled back because it has been marked as rollback-only");
              }
              return;
       }
       ／／处理提交入口
       processCommit(defStatus);
}
```

可以看出rollback和commit都在这个方法中实现。看看 processCommit的实现：

```
private void processCommit(DefaultTransactionStatus status) throws TransactionException {
       try {
              boolean beforeCompletionInvoked = false;
              try {／／事务的提交准备工作由具体的事务处理器来完成
                     prepareForCommit(status);
                     triggerBeforeCommit(status);
                     triggerBeforeCompletion(status);
                     beforeCompletionInvoked = true;
                     boolean globalRollbackOnly = false;
                     if (status.isNewTransaction() || isFailEarlyOnGlobalRollbackOnly()) {
                            globalRollbackOnly = status.isGlobalRollbackOnly();
                     }／／嵌套事务的处理过程。
                     if (status.hasSavepoint()) {
                            if (status.isDebug()) {
                                   logger.debug("Releasing transaction savepoint");
                            }
                            status.releaseHeldSavepoint();
                     }
                     else if (status.isNewTransaction()) {／／根据当前线程中保存的事务状态进行处理，如果当前的事务是一个新的事务，调用具体事务处理器的完成提交，如果当前所持有的事务不是一个新事务，则不提交，由已经存在的事务来完成提交
                            if (status.isDebug()) {
                                   logger.debug("Initiating transaction commit");
                            }
                            doCommit(status);
                     }
                     // Throw UnexpectedRollbackException if we have a global rollback-only
                     // marker but still didn't get a corresponding exception from commit.
                     if (globalRollbackOnly) {
                            throw new UnexpectedRollbackException(
                                          "Transaction silently rolled back because it has been marked as rollback-only");
                     }
              }
              catch (UnexpectedRollbackException ex) {
                     // can only be caused by doCommit
                     triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
                     throw ex;
              }
              catch (TransactionException ex) {
                     // can only be caused by doCommit
                     if (isRollbackOnCommitFailure()) {
                            doRollbackOnCommitException(status, ex);
                     }
                     else {
                            triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
                     }
                     throw ex;
              }
              catch (RuntimeException ex) {
                     if (!beforeCompletionInvoked) {
                            triggerBeforeCompletion(status);
                     }
                     doRollbackOnCommitException(status, ex);
                     throw ex;
              }
              catch (Error err) {
                     if (!beforeCompletionInvoked) {
                            triggerBeforeCompletion(status);
                     }
                     doRollbackOnCommitException(status, err);
                     throw err;
              }

              // Trigger afterCommit callbacks, with an exception thrown there
              // propagated to callers but the transaction still considered as committed.
              try {
                     triggerAfterCommit(status);
              }
              finally {
                     triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
              }

       }
       finally {
              cleanupAfterCompletion(status);
       }
}
```

可以看出，对事务的提交处理都是紧紧围绕TransactionStatus保存的事务处理相关状态进行判断。具体的提交处理过程都设计成抽象方法，交由具体的事务处理器来完成。

#### 事务的回滚

在事务的提交方法中看到了事务的回滚入口，即processRollback方法，其实现代码如下：

```
private void processRollback(DefaultTransactionStatus status) {
       try {
              try {
                     triggerBeforeCompletion(status);
                     if (status.hasSavepoint()) {／／嵌套事务的回滚处理
                            if (status.isDebug()) {
                                   logger.debug("Rolling back transaction to savepoint");
                            }
                            status.rollbackToHeldSavepoint();
                     }／／当前事务调用方法中新建事务的回滚处理
                     else if (status.isNewTransaction()) {
                            if (status.isDebug()) {
                                   logger.debug("Initiating transaction rollback");
                            }
                            doRollback(status);
                     }／／如果在当前事务调用方法中没有新建事务的回滚处理
                     else if (status.hasTransaction()) {
                            if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
                                   if (status.isDebug()) {
                                          logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
                                   }
                                   doSetRollbackOnly(status);
                            }／／由线程的前一个事务来处理回滚，这里不执行任何操作。
                            else {
                                   if (status.isDebug()) {
                                          logger.debug("Participating transaction failed - letting transaction originator decide on rollback");
                                   }
                            }
                     }
                     else {
                            logger.debug("Should roll back transaction but cannot - no transaction available");
                     }
              }
              catch (RuntimeException ex) {
                     triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
                     throw ex;
              }
              catch (Error err) {
                     triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
                     throw err;
              }
              triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
       }
       finally {
              cleanupAfterCompletion(status);
       }
}
```

显然，看了代码我们很快就能理解，Spring 事务传播属性中的 Required_New和NESTED（嵌套事务）的本质区别
1.  PROPAGATION_REQUIRES_NEW 启动一个新的, 不依赖于环境的 "内部" 事务. 这个事务将被完全 commited 或 rolled back 而不依赖于外部事务, 它拥有自己的隔离范围, 自己的锁, 等等. 当内部事务开始执行时, 外部事务将被挂起, 内务事务结束时, 外部事务将继续执行. 
2.  另一方面, PROPAGATION_NESTED 开始一个 "嵌套的" 事务,  它是已经存在事务的一个真正的子事务. 潜套事务开始执行时,它将取得一个 savepoint. 如果这个嵌套事务失败,我们将回滚到此savepoint潜套事务是外部事务的一部分,只有外部事务结束后它才会被提交。
3.  由此可见, PROPAGATION_REQUIRES_NEW 和 PROPAGATION_NESTED 的最大区别在于, PROPAGATION_REQUIRES_NEW 完全是一个新的事务, 而 PROPAGATION_NESTED 则是外部事务的子事务, 如果外部事务 commit, 潜套事务也会被 commit, 这个规则同样适用于 roll back. 

也就是说：
1. PROPAGATION_REQUIRES_NEW事务不受外部事务的影响，是隔离的。
2. PROPAGATION_NESTED，如果内部事务失败且内部，它会回到savepoint之前的状态不会产生脏数据，而外部事务catch住异常后可以选择回滚或者提交；如果外部事务失败，由于嵌套事务是外部事务的一部分，则会导致外部事务与嵌套事务一起回滚。

