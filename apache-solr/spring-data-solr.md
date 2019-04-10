# Spring Data Solr

{% hint style="info" %}
作为背景知识，先学习下 Spring Data repository——用于减少各种数据持久化方案中数据访问层的代码。
{% endhint %}

## 核心概念

### Repository

```java
package org.springframework.data.repository;

import java.io.Serializable;

public interface Repository<T, ID extends Serializable> {

}
```

### CrudRepository

提供 CRUD 操作

```java
public interface CrudRepository<T, ID extends Serializable>
    extends Repository<T, ID> {

    <S extends T> S save(S entity); 

    T findOne(ID primaryKey);       

    Iterable<T> findAll();          

    Long count();                   

    void delete(T entity);          

    boolean exists(ID primaryKey);  

    // … more functionality omitted.
}
```

### PagingAndSortingRepository

除了CRUD，还支持分页和排序

```java
public interface PagingAndSortingRepository<T, ID extends Serializable> 
    extends CrudRepository<T, ID> {

    Iterable<T> findAll(Sort sort);

    Page<T> findAll(Pageable pageable);
}
```

分页示例

```java
PagingAndSortingRepository<User, Long> repository;

// 每页20个User，返回第二页数据
Page<User> users = repository.findAll(new PageRequest(1, 20));
```

## 查询方法

使用 Spring Data 来完成查询，只需要4步

### 创建一个接口

接口应该继承自 Repository 接口

```java
interface PersonRepository extends Repository<Person, Long> 
{ … }
```

### 在接口里声明查询方法

```java
interface PersonRepository extends Repository<Person, Long> {
  List<Person> findByLastname(String lastname);
}
```

### 配置Srping

Spring会生成该接口的代理

```markup
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xmlns:jpa="http://www.springframework.org/schema/data/jpa"
   xsi:schemaLocation="
     http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
     http://www.springframework.org/schema/data/jpa http://www.springframework.org/schema/data/jpa/spring-jpa.xsd">

   <jpa:repositories base-package="com.acme.repositories"/>

</beans>
```

### 注入接口并使用

```java
public class SomeClient {

  @Autowired
  private PersonRepository repository;

  public void doSomething() {
    List<Person> persons = repository.findByLastname("Matthews");
  }
}
```

## 定义 Repostory 接口

### @RepositoryDefinition

继承 CrudRepostory，意味着 CrudRepostory 的方法都被暴露出去；如果只想提供几个有限的方法，可以使用 `@RespotitoryDefinition`，来指定你需要的方法

有选择的 CRUD 方法

```java
@NoRepositoryBean
interface MyBaseRepository<T, ID extends Serializable> extends Repository<T, ID> {

  T findOne(ID id);

  T save(T entity);
}

interface UserRepository extends MyBaseRepository<User, Long> {
  User findByEmailAddress(EmailAddress emailAddress);
}
```

