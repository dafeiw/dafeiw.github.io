---
title: Service Provider Framework
date: 2019-12-12 00:45:21
tags: Java
---

A service provider framework is a system in which multiple service providers implemnet a service, and the system make the implementation available to its clients, decoupling them from the implementation.
<!--more-->
There are four essential components:
- A service interface
- A provider registration API
- A service access API
- A service provider interface (optional)


```java
interface Noodle {
    void addFlour();
}
```

```java
class WhiteNoodle implements Noodle {
    public void addFlour() {
        println("Add white flour")
    }
}
```

```java
class BrownNoodle implements Noodle {
    public void addFlour() {
        println("Add brown flour")
    }
}
```

```java
interface NoodleProvider {
    Noodle getNoodle();
}
```

```java
class WhiteNoodleProvider implements NoodleProvider {

    static {
        NoodleManager.registerProvider("whiteNoodleProvider", new WhiteNoodleProvider())
    }

    public Noodle getNoodle() {
        return new WhiteNoodle();
    }
}
```

```java
class BrownNoodleProvider implements NoodleProvider {

    static {
        NoodleManager.registerProvider("brownNoodleProvider", new WhiteNoodleProvider())
    }

    public Noodle getNoodle() {
        return new BrownNoodle();
    }
}
```

```java
class NoodleManager {

    private static final Map<String, NoodleProvider> providers = new ConcurrentHashMap<String, SaltProvider>();

    public static void registerProvider(String name, SaltProvider p) {  
        providers.put(name, p);  
    }

    public static Noodle getNoodle(String name) {  
        NoodleProvider p = providers.get(name);  
        if (p == null) {  
            throw new IllegalArgumentException(  
                    "No NoodleProvider registered with name:" + name);  
        }  
        return p.getNoodle();  
    }  
}
```

```java
class Test {
    public static void main(String[] args) throws ClassNotFoundException {  
        Class.forName("BrownNoodleProvider");  
        Noodle noodle = NoodleManager.getNoodle("brownNoodleProvider");  
        noodle.addFlour();  
    }
}
```