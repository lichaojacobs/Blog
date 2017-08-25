---
layout: post
title: 手动封装HbaseTemplate mapper类
subtitle: "看spring 源码有感"
date: 2016-10-09 00:24:02
author: "CHAO LI"
tags:
	- Spring
	- Hbase
	- Java
---

## 前言
最近因为业务关系，用到了Hbase，因为用的是Spring boot框架 ，所以自然而然就用到了spring封装的HbaseTemplate工具类。然而HbaseTemplate封装的代码实在比较糟糕，出了一些基本的CRUD操作之外并没有给我们提供太多便利之处。先来看看痛处：

### 痛处一及改进 ###

 - 我们先来看看HabaseTemplate最基本的查询操作(以下只是demo演示)：
 
```
 class UserInfo{
	 string name;
	 string password;
 }
 public void putUserInfo(UserInfo userInfo) {
    hBaseTemplate.execute(TABLE_NAME, (table) -> {
      //根据rowKey定义一个put对象，可用作插入和更新
      Put put = new Put(Bytes.toBytes(rowKey));
     //name是否为空
      if(userInfo.name!=null){
      put.addColumn(COLUMN_FAMILY_NAME.getBytes(), Bytes.toBytes(COLUMN_RAW_DATA)，Bytes.toBytes(userInfo.name));
      }
      //password是否为空
      if(userInfo.password!=null){
      put.addColumn(COLUMN_FAMILY_NAME.getBytes(), Bytes.toBytes(COLUMN_RAW_DATA)，Bytes.toBytes(userInfo.password));
      }
      table.put(put);
      return true;
    });
  }

```
相信大家也看出来了，如果待插入的对象有很多字段呢？还要逐个写if语句来判读非空么？这明显使得代码非常地不简洁。于是，个人封装了一个插入更新模版类（其实只是简单的对Put对象的一个扩展）：


```
//继承并扩展Put对象
public class PutExtension extends Put {

  String columnFamilyName = "demo";

  public PutExtension(String columnFamilyName, byte[] row) {
    super(row);
    this.columnFamilyName = columnFamilyName;
  }
  
  public PutExtension build(String paramName, Object param) throws IOException {
    if (param != null) {
      this.addColumn(columnFamilyName.getBytes(), paramName.getBytes(),
          Bytes.toBytes(param.toString()));
    }
    return this;
  }
}

```

封装之后，之前累赘的查询操作可以变得如下所示：

``` 

//然后操作如下
hBaseTemplate.execute(TABLE_NAME, (table) -> {
      PutExtension putExtension = new PutExtension(familyName, rowKey.getBytes());
      putExtension.build("name",userInfo.name)
          .build("password", userInfo.password);
      table.put(putExtension);
      return true;
    })

```
其实这也就是一个简单的封装，只不过把冗余的逻辑判断给丢出去了而已。

### 痛处二及改进 ###

 - 在HbaseTemplate中，根据rowKey查询出来的原始数据是字节数组，我们要将字节数组转化成业务逻辑中希望的java bean需要做很多重复的判断匹配逻辑，以下是没改进前的代码：
 
```

 public UserInfo getUserInfo() {
    return (UserInfo) hBaseTemplate.get(TABLE_NAME, rowKey, familyName,(result, i) ->{
     UserInfo userInfo=new UserInfo()
     //重复逻辑一
bytes[] nameBytes=result.getValue(familyName.getBytes(), "name".getBytes()));
if(nameBytes!=null){
  userInfo.setName(Bytes.toString(nameBytes));
}
//重复逻辑二
bytes[] passwordBytes=result.getValue(familyName.getBytes(), "password".getBytes()));
if(passwordBytes!=null){
  userInfo.setPassword(Bytes.toString(passwordBytes));
}
  });
}

```

可以看出，这样做的缺点是一旦java bean的字段一多，重复的非空判断逻辑也会增多，从而使得代码变得十分累赘且不可维护。于是我参考Spring JDBC的RowMapper的封装，利用了Spring框架自带的反射工具beanUtils和beanWrapper，自己实现了如下封装：

``` 
public class HBaseResultBuilder<T> {
  private Class<T> mappedClass;
  private Map<String, PropertyDescriptor> mappedFields;
  private Set<String> mappedProperties;
  HashSet populatedProperties;
  private BeanWrapper beanWrapper;
  private Result result;
  private String columnFamilyName;
  private T t;
  //接受一些列参数并实例化要返回的结果对象
  public HBaseResultBuilder(String columnFamilyName, Result result, Class<T> clazz) {
    this.columnFamilyName = columnFamilyName;
    this.result = result;
    this.mappedClass = clazz;
    mappedFields = new HashMap<>();
    mappedProperties = new HashSet<>();
    populatedProperties = new HashSet<>();
    this.t = BeanUtils.instantiate(clazz);
    PropertyDescriptor[] pds = BeanUtils.getPropertyDescriptors(mappedClass);
    PropertyDescriptor[] var3 = pds;
    int var4 = pds.length;
    for (int var5 = 0; var5 < var4; ++var5) {
      PropertyDescriptor pd = var3[var5];
      if (pd.getWriteMethod() != null) {
        this.mappedFields.put(this.lowerCaseName(pd.getName()), pd);
        String underscoredName = this.underscoreName(pd.getName());
        if (!this.lowerCaseName(pd.getName()).equals(underscoredName)) {
          this.mappedFields.put(underscoredName, pd);
        }
        this.mappedProperties.add(pd.getName());
      }
    }
    beanWrapper = PropertyAccessorFactory.forBeanPropertyAccess(t);
  }

  private String underscoreName(String name) {
    if (!StringUtils.hasLength(name)) {
      return "";
    } else {
      StringBuilder result = new StringBuilder();
      result.append(this.lowerCaseName(name.substring(0, 1)));

      for (int i = 1; i < name.length(); ++i) {
        String s = name.substring(i, i + 1);
        String slc = this.lowerCaseName(s);
        if (!s.equals(slc)) {
          result.append("_").append(slc);
        } else {
          result.append(s);
        }
      }

      return result.toString();
    }
  }

  private String lowerCaseName(String name) {
    return name.toLowerCase(Locale.US);
  }
  //使用时根据要解析的字段频繁调用此方法即可，仿造java8 流式操作
  public HBaseResultBuilder build(String columnName) {
    byte[] value = result.getValue(columnFamilyName.getBytes(), columnName.getBytes());
    if (value == null || value.length == 0) {
      return this;
    } else {
      String field = this.lowerCaseName(columnName.replaceAll(" ", ""));
      PropertyDescriptor pd = this.mappedFields.get(field);
      if (pd == null) {
        log.error("HBaseResultBuilder error: can not find property: " + field);
      } else {
        beanWrapper.setPropertyValue(pd.getName(), Bytes.toString(value));
        populatedProperties.add(pd.getName());
      }
    }
    return this;
  }
  
  //伪造Java8的即视感，“流最后的终端操作“。
  public T fetch() {
    //只要有一个属性被解析出来就返回结果对象，毕竟hbase存的是稀疏数据，不一定全量
    if (CollectionUtils.isNotEmpty(populatedProperties)) {
      return this.t;
    } else {
      return null;
    }
  }

```

通过利用反射的基本原理，我们可以通过结果数据构造出我们需要的java bean。最后我们的调用过程可以简化成如下：

``` 

public UserInfo getUserInfo() {
    return (UserInfo) hBaseTemplate.get(TABLE_NAME, rowKey,    familyName,
        (result, i) -> new HBaseResultBuilder<>(familyName, result, UserInfo.class).build("name").build("password").fetch());
  }

```
成功！！！是不是代码整洁多了，其实也就是将一些复杂的逻辑给抽出去了，正好最近看了Java8实战，从而萌生的一点小想法。