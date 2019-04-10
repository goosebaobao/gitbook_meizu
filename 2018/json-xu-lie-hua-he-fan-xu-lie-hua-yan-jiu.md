# Json 序列化和反序列化研究

## 概念

* json 序列化：把 javabean 转换成 json 字符串
* json 反序列化：把 json 字符串转换成对应的 javabean

## 问题

1. 序列化时，是调用 getXxx 方法来获取 xxx 属性的值吗？
2. 反序列化时，是调用 setXxx 方法来设置 xxx 属性的值吗？

## 试验

我们用一个 People 类来做试验，注意 43 行的代码

```java
public class People {
    /** 姓名 **/
    private String name;
    /** 身高 m **/
    private double height;
    /** 体重 kg **/
    private double weight;
    /** bmi 指数 **/
    private double bmi;


    public String getName() {
        return name;
    }


    public void setName(String name) {
        this.name = name;
    }


    public double getHeight() {
        return height;
    }


    public void setHeight(double height) {
        this.height = height;
    }


    public double getWeight() {
        return weight;
    }


    public void setWeight(double weight) {
        this.weight = weight;
    }


    public double getBmi() {
        bmi = weight / (height * height);
        return bmi;
    }


    public void setBmi(double bmi) {
        this.bmi = bmi;
    }

}
```

### fastjson

```java
@Test
public void fastjson() {
    People people = new People();
    people.setName("八戒");
    people.setWeight(100.00);
    people.setHeight(1.70);
    people.setBmi(30);

    log.info(JSON.toJSONString(people));

}
```

输出结果为

```javascript
{"bmi":34.602076124567475,"height":1.7,"name":"八戒","weight":100.0}
```

可见，fastjosn 是调用了 getXxx 方法来获取属性值

### gson

```java
@Test
public void gson() {
    People people = new People();
    people.setName("八戒");
    people.setWeight(100.00);
    people.setHeight(1.70);
    people.setBmi(30);

    log.info(new Gson().toJson((people)));

}
```

输出结果为

```javascript
{"name":"八戒","height":1.7,"weight":100.0,"bmi":30.0}
```

可见，gson 是直接获取 xxx 属性的值

## 试验二

我们修改一下 People 类，删除 bmi 属性，仅保留 getBmi\(\) 方法，如下

```java
public class People {
    /** 姓名 **/
    private String name;
    /** 身高 m **/
    private double height;
    /** 体重 kg **/
    private double weight;


    public String getName() {
        return name;
    }


    public void setName(String name) {
        this.name = name;
    }


    public double getHeight() {
        return height;
    }


    public void setHeight(double height) {
        this.height = height;
    }


    public double getWeight() {
        return weight;
    }


    public void setWeight(double weight) {
        this.weight = weight;
    }


    public double getBmi() {
        return weight / (height * height);

    }

}
```

### fastjson

```java
@Test
public void fastjson() {
    People people = new People();
    people.setName("八戒");
    people.setWeight(100.00);
    people.setHeight(1.70);
    // people.setBmi(30);

    log.info(JSON.toJSONString(people));

}
```

输出结果为

```javascript
{"bmi":34.602076124567475,"height":1.7,"name":"八戒","weight":100.0}
```

可见，即使不存在属性，但只要有 get 方法，fastjosn 会视同有这个属性

### gson

```java
@Test
public void gson() {
    People people = new People();
    people.setName("八戒");
    people.setWeight(100.00);
    people.setHeight(1.70);
    // people.setBmi(30);

    log.info(new Gson().toJson((people)));

}
```

输出结果为

```javascript
{"name":"八戒","height":1.7,"weight":100.0}
```

可见，gson 不会根据 get 方法来推断属性

## fastjson 序列化和反序列化的风险

由于 fastjson 是根据 get 方法来推断属性并取值，就存在这样的风险：一个对象，经过 fastjson 序列化再反序列化以后，会得到一个不相等\(equals\)的对象，看下面代码

```java
public class Car {

    private String brand;
    private long   speed;


    public String getBrand() {
        return brand;
    }


    public void setBrand(String brand) {
        this.brand = brand;
    }


    public long getSpeed() {
        if (speed < 100) {
            speed = 100;
        } else {
            speed = 200;
        }
        return speed;
    }

}
```

测试用例如下

```java
@Test
public void test() {
    Car car = new Car();
    car.setBrand("bmw");
    String json = JSON.toJSONString(car);
    log.info(json);
    Car car1 = JSON.parseObject(json, Car.class);
    Assert.assertNotEquals(car.getSpeed(), car1.getSpeed());

}
```

运行测试，发现两个对象并不相等

## 结论

虽然上面仅对 get 方法进行了试验，没有试验 set 方法，但应该不影响结论——无论是用 fastjson 还是 gson 对一个 java 对象进行序列化/反序列化，都要确保以下 2 点 

1. get/set 方法对应的属性存在 
2. 属性值和通过 get 方法得到的值相同：即在 get/set 方法里不要做额外的操作 

如果上面 2 点满足不了，可能会出现莫名其妙的问题

## 思考

### 什么是 javabean

到底什么是 javabean？wikipedia 的 [定义](https://en.wikipedia.org/wiki/JavaBeans) 如下

> In computing based on the Java Platform, JavaBeans are classes that encapsulate many objects into a single object \(the bean\). They are serializable, have a zero-argument constructor, and allow access to properties using getter and setter methods

关于属性值和 get/set 方法是这样的

> access to properties using getter and setter methods

根据这个说法，我认为 get/set 方法是用来访问属性值的，这意味着 

1. 如果没有这个属性，就不该有 get/set 方法
2. get/set 方法应该是获取/设置属性值，而不包含其他功能

### gson 和 fastjson 的处理对象

先看 [gson](https://github.com/google/gson)

> A Java serialization/deserialization library to convert Java Objects into JSON and back 
>
> Gson is a Java library that can be used to convert Java Objects into their JSON representation. It can also be used to convert a JSON string to an equivalent Java object. Gson can work with arbitrary Java objects including pre-existing objects that you do not have source-code of.

gson 是对 java 对象进行序列化和反序列化的库，并没有限定在 javabean 上，我的想法是（未验证） 

1. gson 默认是对 javabean 进行操作并遵循 javabean 的规范
2. gson 也可以操作非 javabean 的对象，但该对象要通过特殊的注解来标记需要处理的属性或方法

再看 [fastjson](https://github.com/alibaba/fastjson)

> A fast JSON parser/generator for Java
>
> Fastjson is a Java library that can be used to convert Java Objects into their JSON representation. It can also be used to convert a JSON string to an equivalent Java object. Fastjson can work with arbitrary Java objects including pre-existing objects that you do not have source-code of.

居然和 gson 一毛一样，也不知道谁抄谁的

## 相关规范

由于不同的 json 库实现上存在差异，所以我们需要定义出一个规范来：json 库用哪一个？

这里明确规定 

1. json 库使用 fastjson，版本尽量用最新版
2. 凡是需要用 fastjson 进行序列化和反序列化的 java 对象，必须是 javabean——也就是说，不得在 get/set 方法里执行获取/设置属性值以外的代码

