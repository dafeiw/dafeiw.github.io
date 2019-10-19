---
title: Strategy Pattern
date: 2019-08-26 20:41:44
tags:
categories: Design Pattern
---

### Introduction

Strategy Pattern can be used to avoid if-else or swtich statement.

Define a family of algorithms, encapsulate each one, and make them interchangeable. 

Encapsulate interface details in a base class, and bury implementation details in derived classes
<!--more -->
{% asset_img Strategy1.png strategy %}

The client is compled only to an abstraction.

1. Identify an algorithm
2. Specify the signature for that algorithm in an interface
3. Implement the interface for concrete algorithm
4. Clients of the algorithm uses the interface.

### Example:

* Interface:

```java
public interface PayStrategy {
    public void pay(float money);
}
```

* Client: 

```java
public class ContextStrategy {
    private PayStrategy strategy;
    public ContextStrategy(){

    }
    public void payout(float money){
        strategy.pay(money);//调用策略
    }

    public void setStrategy(PayStrategy strategy){
        this.strategy=strategy;//设置策略类
    }
}
```

* Concrete strategies:

```java
public class AliPayStrategy implements PayStrategy {

    @Override
    public void pay(float money) {
        //调用支付宝的接口具体代码 略...
        if(money<=200){
            // ...
        }else{
            // ...
        }
    }

}
```

```java
public class WeChatPayStrategy implements PayStrategy {

    @Override
    public void pay(float money) {
        // ...
    }

}
```