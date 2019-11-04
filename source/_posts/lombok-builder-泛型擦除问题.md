---
title: lombok builder 泛型擦除
date: 2017-09-23 21:51:48
tags:
    - Java
    - 成长
---

### 前言

- 众所周知，Java长期以来比较遭业界嫌弃的是太笨重，代码冗余过大。然而依托于Java庞大健全的开源社区，这些缺点正在逐渐改善。Java 8 引进的lambda以及函数式编程的思想让我们的代码越来越简洁。lombok等各大开源神器让我们的冗余代码越来越少。

- 使用lombok已经很长时间了，一直很好用，然而最近发现使用lombok builder构造泛型的时候会出现泛型擦除的情况，导致得到的对象是Object类型

### 案例

- 定义一个待使用builder构造的泛型类

```
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class PageableResponse<T> {

  List<T> results;
  @SerializedName("total_pages")
  long totalPages;
  @SerializedName("total_elements")
  long totalElements;
}

```

正常情况下，我是可以这样使用的:

```
  public PageableResponse<String> getPageableResponse() {
    return PageableResponse
        .builder()
        .results(Lists.newArrayList())
        .totalElements(0)
        .totalPages(0)
        .build();
  }

```

然而出现类型转换错误，PageableResponse.builder()生成的是PageableResponse.PageableResponseBuilder`<Object>` 想来也是，我这里调用builder方法时候并没有指定任何类型。

### 泛型基础知识

那为什么会得到这样的结果？这里就要回顾一下泛型的知识了，这里我们的场景比较复杂一点，属于泛型类里面的静态泛型方法。这里提一下知识点：

##### java泛型转换的几个事实：

- 虚拟机中没有泛型，只有普通的类和方法
- 无论何时定义一个泛型类型，都自动提供了一个相应的原始类型。原始类型的名字就是删去类型参数后的泛型类型名。擦除类型变量，替换为限定类型（无限定的变量用Object）。当程序调用泛型方法时，如果擦除返回类型，编译器插入强制类型转换
- 所有的类型参数都用它们的限定类型替换 如:Pair`<T>` 擦除类型之后就变成了Pair`<Object>`
- 桥方法被合成来保持多态
- 为保证类型安全性，必要时插入强制类型转换

##### 泛型的约束与局限性

- 不能用基本类型实例化类型参数

- 运行时类型查询只适用于原始类型:
	- if(a instanceof Pair`<String>`)//ERROR
	- 同理getClass方法也总是返回原始类型:

		```
		Pair<String> stringPair=...
		Pair<Employ> employPair=...
		if(stringPair.getClass()==employPair.getClass()) //True
		//两次调用都将返回Pair.class

		```
- 不能创建参数化类型数组
	
	```
	Pair<String>[] table = new Pair<String>[10]//ERROR
	//类型擦除之后，table的类型是Pair[]。可以把它转换为Object[]
	Object[] arr = table
	//数组会记住它的元素类型，如果试图存储其他类型的元素，会抛出ArrayStoreException异常
	//只是不允许创建这些数组，但是声明类型为Pair<String>[]的变量仍是合法的，只是不能用new Pair<String>[10]初始化这个变量
	//可以声明通配类型的数组，然后进行类型转换，导致的结果将是不安全的
	Pair<String>[] table = (Pair<String>[]) new Pair<?>[10];

	```

- 不能实例化类型变量: 不能new T(....)

	- 类型擦除会将T改变成Object，而且，本意肯定不希望调用new Object()
	- 可以通过反射调用Class.newInstance方法来构造泛型对象，细节有点复杂

		```
		//不能以下方式调用
		first = T.class.newInstance();//ERROR
		//表达式T.class是不合法的，必须如下方式才可以支配Class对象：

		public static<T> Pair<T> makePair(Class<T> cl){
		   try { return new Pair<>(cl.newInstance(),cl.newInstance()) }
		   catch (Exception ex) { return null; }
		}

		Pair<String> p = Pair.makePair(String.class);
		//注意，Class类本身就是泛型，String.class是一个Class<String>的实例（唯一实例）。因此makePair方法能够判断出pair的类型。

		```
- 不能以如下的方式构造泛型数组

	```
	//类型擦除会让这个方法永远构造Object[2]数组，而要求extends Comparable，这显然会抛出转换异常
	// 此时可以把 extends Comparable去掉，这样能保证强制转换的正确性
	public static <T extends Comparable> T[] minmax(T... a) {

    	Object[] mm = new Object[2];
    	mm[0] = a[0];
    	mm[1] = a[1];

    	return (T[]) mm;
  	}

  	或者使用反射，调用Array.newInstance:

  	public static <T extends Comparable> T[] minmax(T... a) {

    	T[] mm = (T[]) Array.newInstance(a.getClass().getComponentType(), 2);
    	...
  	}


	```

##### 泛型类里面的泛型方法和静态泛型方法是有区别的：

- 泛型类定义的泛型 在整个类中有效 如果被方法使用，当泛型类确定类型之后，泛型方法也就确定类型了
- 而对于静态泛型方法而言，其泛型的类型是不依赖于泛型类的类型，也就是这两者的类型完全不相干（依据上一条的事实一）

##### 泛型类型的继承规则

- 考虑一个类和一个子类，如Employee和Manager。Pair`<Manager>` 却不是Pair`<Employee>`的子类。事实上，它们的关系如下图所示

![](http://jacobs.wanhb.cn/images/generics1.png)

- 泛型类可以扩展或实现其他泛型类。这一点与普通的类没什么区别。如：ArrayList`<T>` 类实现List`<T>`接口，意味着一个ArrayList`<Manager>` 可以被转换为一个List`<Manager>`。但是一个ArrayList`<Manager>` 不是一个ArrayList`<Employee>` 或者List`<Employee>`，它们的关系如下图:

![](http://jacobs.wanhb.cn/images/generics2.png)


###  案例解释

- 泛型知识作了一番复习之后，我们再回到上面的案例。之所以无法正常的使用builder，是因为静态泛型方法不继承泛型类的类型。下面是案例类生成的代码：

```
public class PageableResponse<T> {
  List<T> results;
  @JSONField(
    name = "total_pages"
  )
  long totalPages;
  @JSONField(
    name = "total_elements"
  )
  long totalElements;

  //这个T不依赖于PageableResponse<T>的T
  public static <T> PageableResponse.PageableResponseBuilder<T> builder() {
    return new PageableResponse.PageableResponseBuilder();
  }
  
  ...
  }

```

- 而正常的没有泛型的生成代码如下：

```

public class Address {
  private String city;

  //不带泛型
  public static Address.AddressBuilder builder() {
    return new Address.AddressBuilder();
  }

  public Address() {
  }

  @ConstructorProperties({"city"})
  public Address(String city) {
    this.city = city;
  }

  public static class AddressBuilder {
    private String city;

    AddressBuilder() {
    }

    public Address.AddressBuilder city(String city) {
      this.city = city;
      return this;
    }

    public Address build() {
      return new Address(this.city);
    }

    public String toString() {
      return "Address.AddressBuilder(city=" + this.city + ")";
    }
  }
}

```

- 所以如果必须要使用泛型且如果这种情况不是太多的话，可以自己实现lombok的builder生成的代码，以案例定义的类为例，改造完的代码如下：

```

@NoArgsConstructor
@AllArgsConstructor
public class PageableResponse<T> {

  List<T> results;
  @SerializedName("total_pages")
  long totalPages;
  @SerializedName("total_elements")
  long totalElements;

  //这里使用非静态方法的写法可以达到继承泛型类型的目的
  public PageableResponseBuilder<T> builder() {
    return new PageableResponseBuilder<T>();
  }

  public class PageableResponseBuilder<T> {

    List<T> results;
    long totalPages;
    long totalElements;

    public PageableResponseBuilder<T> totalPages(long totalPages) {
      this.totalPages = totalPages;
      return this;
    }

    public PageableResponseBuilder<T> totalElements(long totalElements) {
      this.totalElements = totalElements;
      return this;
    }

    public PageableResponseBuilder<T> results(List<T> results) {
      this.results = results;
      return this;
    }

    public PageableResponse<T> build() {
      return new PageableResponse<>(results, totalPages, totalElements);
    }
  }
}

```



