---
title: 类函数式的sql生成工具类的封装
date: 2017-01-30 00:54:43
tags: java8 sqlGenerator 函数式
---

### 前言
自从正式工作以来，公司一直用的是Spring 原生的JDBC Template以及在其上封装的扩展的一些小工具， 而摈弃了Mybatis、ibatis等ORM框架。总的来说，这种做法对于开发效率来说提高不少，由于真正的查库操作不会直接穿透到Mysql，所以抗压性也没有太大的问题。

### 问题
虽说直接使用原生的JDBC Template比较方便，但是构造参数条件，以及生成查询语句过于简单粗暴，导致代码不简洁，复用度也不高。举个例子，根据特定的条件查询用户：
```
    StringBuffer sqlBuilder = new StringBuffer("select * from user where ");
    Map<String, Object> paramsMap = new HashMap<>();
    if (userId != null) {
      sqlBuilder.append("user_id=:userId");
      paramsMap.put("userId", userId);
    }
    if (batchNumber != null) {
      sqlBuilder.append("batch_number=:batchNumber");
      paramsMap.put("batchNumber", batchNumber);
    }
    if (status != null) {
      sqlBuilder.append("status=:status");
      paramsMap.put("stats", status);
    }

    if (page > 0 && count > 0) {
      sqlBuilder.append(" limit ")
          .append(page * count)
          .append(", ")
          .append(count);
    }
    //生成sql语句
    sqlBuilder.toString();
    
```
可以看到上面的代码大部分在重复同样的逻辑，20多行的代码仅仅只是在构造sql语句以及收集参数，而且这些重复的代码将充斥项目所有的DAO层，导致代码非常不整洁。维护困难。

### sql生成工具类封装
秉承着恶心重复的代码要重构抽象的态度，对sql生成的步骤，做了一次简单的封装：

- 先介绍接口
```
public interface Order {
    Where desc();

    Where asc();
  }

public interface Where {
    Where ifPresent(Object value, String sql);

    Order orderBy(String field);

    Where limit(int begin, int end);

    Where in(List<Object> values, String sql);

    String sql();//构造的sql

    Map<String, Object> params();//参数值
  }
```
根据sql语句的特点，抽出子句部分的生成单独构造，涵盖Where, in(暂未实现)，limit 以及 order 排序规则。

- 来看看接口的实现
将所有相关的类通过内部静态类封装入SqlWhereBuffer类中。
```
public class SqlWhereBuffer {

  public static SqlWhereBuffer.Where builder() {
    return new Builder();
  }

  static class Builder implements SqlWhereBuffer.Order, SqlWhereBuffer.Where {
    private List<String> sql = new ArrayList<>();
    private Map<String, Object> params = new HashMap<>();
    private String orderBy = " ";
    private String limit = " ";
    private String in = " ";
    private List<SqlMapperPlugin> sqlMapperPlugins;

    public Builder() {
      //添加基本类型默认的插件
      sqlMapperPlugins = Lists.newArrayList();
      this.mapperPlugins(SqlMapperPlugin.EnumSqlPlugin)
          .mapperPlugins(SqlMapperPlugin.BooleanSqlPlugin);
    }

    public Builder mapperPlugins(SqlMapperPlugin sqlMapperPlugin) {
      if (CollectionUtils.isEmpty(sqlMapperPlugins)) {
        sqlMapperPlugins = Lists.newArrayList();
      }
      sqlMapperPlugins.add(sqlMapperPlugin);
      return this;
    }

    @Override
    public SqlWhereBuffer.Where ifPresent(Object value, String sql) {
      Optional.ofNullable(value)
          .ifPresent(val -> {
            this.sql.add(sql);
            Pattern pattern = Pattern.compile(":([a-z,A-Z,\\w,_]*)"); //固定的sql参数模式
            Matcher matcher = pattern.matcher(sql);
            if (!matcher.find()) {
              throw new IllegalArgumentException(sql + " don't include :name");
            }

            Optional<SqlMapperPlugin> mapper = sqlMapperPlugins.stream()
                .filter(sqlMapperPlugin -> sqlMapperPlugin.test(val))
                .findFirst();
            if (mapper.isPresent()) {
              params.putAll(mapper.get()
                  .getParams(matcher.group(1), value));
            } else {
              //匹配不到走默认的插件
              params.put(matcher.group(1), value);
            }
          });
      return this;
    }

    @Override
    public SqlWhereBuffer.Order orderBy(String field) {
      orderBy += "ORDER BY " + field;
      return this;
    }

    @Override
    public SqlWhereBuffer.Where limit(int offset, int count) {
      limit += "limit " + offset + "," + count;
      return this;
    }
    
    @Override
    public String sql() {
      if (sql.isEmpty() || params.isEmpty()) {
        return "";
      } else {
        return " WHERE " + sql.stream()
            .collect(Collectors.joining(" AND ")) + orderBy + limit;
      }
    }

    @Override
    public ImmutableMap<String, Object> params() {
      return ImmutableMap.copyOf(params);
    }

    @Override
    public SqlWhereBuffer.Where desc() {
      orderBy += " DESC";
      return this;
    }

    @Override
    public SqlWhereBuffer.Where asc() {
      return this;
    }
  }
}
```
静态类Builder实现了SqlWhereBuffer.Order, SqlWhereBuffer.Where这俩个接口的方法。ifPresent是主入口。流程如下：
```
1、在ifPresent中参数先进行非空判断，如果是空则直接过滤掉；
2、然后通过一个正则表达式（**:([a-z,A-Z,\\w,_]*)**）对参数sql进行规则校验，如果不符合也将过滤；
3、最后value通过一系列的自定义的注册插件的匹配判断，得出sql以及params。
```
- 关于自定义插件，本没想做这么复杂，然而在实际的使用过程中，JDBC Template对Enum的支持不够...不过想也是，Enum大多是自定义的，没法做到一套约定的接口满足所有需求。于是封装了一个插件类，方便以后扩展：

```
public static class SqlMapperPlugin {
    private final Predicate<Object> predicate;
    private final ParamValue paramValue;

    private SqlMapperPlugin(Predicate<Object> predicate,
        ParamValue paramValue) {
      this.predicate = predicate;
      this.paramValue = paramValue;
    }

    //定义枚举插件
    static SqlMapperPlugin EnumSqlPlugin = of(Enum.class).paramValue((value, sql) ->
        Collections.singletonMap(sql, getEnumValue(value))
    );
    //定义Boolean插件
    static SqlMapperPlugin BooleanSqlPlugin = of(Boolean.class).paramValue((value, sql) ->
        Collections.singletonMap(sql, ((boolean) value) ? 1 : 0)
    );

    public static SqlMapperPlugin.MapperPluginsBuilder of(Predicate<Object> predicate) {
      return new MapperPluginsBuilder(predicate);
    }

    public static SqlMapperPlugin.MapperPluginsBuilder of(Class clazz) {
      return of((pd) -> {
        return clazz.isAssignableFrom(pd.getClass());
      });
    }

    boolean test(Object pd) {
      return this.predicate.test(pd);
    }

    Map<String, Object> getParams(String sql, Object value) {
      return paramValue.getParams(value, sql);
    }

    public static class MapperPluginsBuilder {

      Predicate<Object> predicate;

      public MapperPluginsBuilder(Predicate<Object> predicate) {
        this.predicate = predicate;
      }

      public SqlMapperPlugin paramValue(SqlMapperPlugin.ParamValue paramValue) {
        return new SqlMapperPlugin(this.predicate, paramValue);
      }
    }

    @FunctionalInterface
    public interface ParamValue {
      Map<String, Object> getParams(Object var1, String var2);
    }
  }

  public static String getEnumValue(Object value) {
    Method method;
    try {
      method = value.getClass()
          .getMethod("name");
      return String.valueOf(method.invoke(value));
    } catch (NoSuchMethodException e) {
      throw new RuntimeException("NoSuchMethodException", e);
    } catch (IllegalAccessException e) {
      throw new RuntimeException("IllegalAccessException", e);
    } catch (InvocationTargetException e) {
      throw new RuntimeException("InvocationTargetException", e);
    }
  }
```
主要用到了java8中新增的Predicate接口，用于判断value的Class类型，以及自己定义了一个FunctionalInterface 用于获取查询参数。这里先实现了EnumSqlPlugin，BooleanSqlPlugin俩个插件，之后有其他类型的需求，也可以通过类似的形式加入。对EnumSqlPlugin实现进行解释：
```
//定义枚举插件
    static SqlMapperPlugin EnumSqlPlugin = of(Enum.class).paramValue((value, sql) ->
        Collections.singletonMap(sql, getEnumValue(value))
    );
```
通过定义的of方法定义赋值Predicate,作为判断value是否为指定类型的方法。
```
public static SqlMapperPlugin.MapperPluginsBuilder of(Class clazz) {
      return of((pd) -> {
        return clazz.isAssignableFrom(pd.getClass());
      });
    }
```
of方法返回了MapperPluginsBuilder类，紧接着定义函数式接口paramValue的实现：
```
Collections.singletonMap(sql, getEnumValue(value))
```
最后返回EnumSqlPlugin对应的新实例:
```
return new SqlMapperPlugin(this.predicate, paramValue);
```
使用的时候，在Builder的构造函数中注入默认的插件即可，如日后有扩展，也可调用mapperPlugins方法动态加入：
```
 private List<SqlMapperPlugin> sqlMapperPlugins;
 public Builder() {
      //添加基本类型默认的插件
      sqlMapperPlugins = Lists.newArrayList();
      this.mapperPlugins(SqlMapperPlugin.EnumSqlPlugin)
          .mapperPlugins(SqlMapperPlugin.BooleanSqlPlugin);
    }

public Builder mapperPlugins(SqlMapperPlugin sqlMapperPlugin)    {
      if (CollectionUtils.isEmpty(sqlMapperPlugins)) {
        sqlMapperPlugins = Lists.newArrayList();
      }
      sqlMapperPlugins.add(sqlMapperPlugin);
      return this;
    }
```

### 工具的使用
现在，我们可以使用封装好的工具了:
```
SqlWhereBuffer.Where where = SqlWhereBuffer.builder()
        .ifPresent(batchNumber, "batch_number=:batchNumber")
        .ifPresent(status, "upload_result=:uploadResult")
        .ifPresent(userId, "user_id=:userId")
        .limit(page * count, count)
        .orderBy("id")
        .asc();
```
这样，代码变得整洁多了，也符合java8函数式风格的效果。

