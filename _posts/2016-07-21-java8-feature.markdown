---
layout: post
title:  "Java8 新特性"
date:   2016-07-21 09:30
categories: coding
tags: coding
excerpt: Android N版本正式支持Java8特性，主要包括默认接口方法，Lambda表达式，重复注解和方法引用等特性。下面主要介绍这4个特性以及运行环境依赖。   
---


### 默认方法和静态方法
默认方法主要用于接口的扩展，在以下场景中，各个地区通过TimeClient获取当地时间。    

```java
public interface TimeClient {
    void setTime(int hour, int minute, int second);
    void setDate(int day, int month, int year);
    LocalDateTime getDateTime();
}
```

当需求需要扩展时，例如本地需要获取其他地区的时间，在Java8之前我们有以下方法来解决:  

 * 在TimeClient类中增加getZonedDateTime(String zone)接口，并让各个地区的TimeClient类实现此接口。  
缺点：实现代码都是一样的，代码重复严重，且需要修改所有子类。  


 * 通过装饰模式包装TimeClient类，实现获取其他地区时间的功能。  
缺点：会产生很多组合类  

```java
public interface WuhanInBeijingTimeClient extends WuhanTimeClient {
    TimeClient timeClient;
    public WuhanInBeijingTimeClient (TimeClient client) {
        timeClient = client; 
    }
    @Override
    public void getDateTime() {
        return timeClient.getDateTime();
    }
}
```

当老版本代码需要扩展功能时，这种场景是很常见的，使用Java8的默认方法，可以很好的解决此问题。

```java
interface TimeClient {
    public interface TimeClient {
        void setTime(int hour, int minute, int second);
        void setDate(int day, int month, int year);
        LocalDateTime getDateTime();
        default ZonedDateTime getZonedDateTime(String zoneString) {
            return ZonedDateTime.of(getLocalDateTime(), getZoneId(zoneString));
        }
    }
}
```

Java语法中一个类智能继承自一个父类，并实现多个接口。既然Java8中实现方法是可行的，接口方法的多实现也是可行的。以下规则决定了哪个方法会被调用。
规则一：定义在类中的方法优先于定义在接口的方法：

```java
interface A {
  default void doSth(){
  System.out.println("inside A");
  }
}
class App implements A{

  @Override
  public void doSth() {
    System.out.println("inside App");
  }

  public static void main(String[] args) {
    new App().doSth();
  }
}
```
运行结果：inside App。调用的是在类中定义的方法。

规则二：否则，调用定制最深的接口中的方法。

```java
interface A {
  default void doSth() {
    System.out.println("inside A");
  }
}
interface B {}
interface C extends A {
  default void doSth() {
    System.out.println("inside C");
  } 
}
class App implements C, B, A {

  public static void main(String[] args) {
    new App().doSth();
  }
}
```
运行结果：inside C

规则三：否则，直接调用指定接口的实现方法

```java
interface A {
  default void doSth() {
    System.out.println("inside A");
  }
}
interface B {
  default void doSth() {
    System.out.println("inside B");
  }
}
class App implements B, A {

  @Override
  public void doSth() {
    B.super.doSth();
  }

  public static void main(String[] args) {
      new App().doSth();
  }
}
```
运行结果： inside B

在接口中还可以定义静态方法，每个类的实例都共享接口的静态方法，这样我们不再需要在一个实体类中定义静态方法。

```java
public interface TimeClient {
    static public ZoneId getZoneId (String zoneString) {
        try {
            return ZoneId.of(zoneString);
        } catch (DateTimeException e) {
            System.err.println("Invalid time zone: " + zoneString +
                "; using default time zone instead.");
            return ZoneId.systemDefault();
        }
    }

    default public ZonedDateTime getZonedDateTime(String zoneString) {
        return ZonedDateTime.of(getLocalDateTime(), getZoneId(zoneString));
    }    
}
```
