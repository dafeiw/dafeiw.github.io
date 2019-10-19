---
title: Springboot Controller Method Summary
date: 2019-08-02 22:17:18
tags:
- spring
categories: Spring
---

# RequestParam

## defaultValue

```java
    @RequestMapping("/person")
    public String hello(@RequestParam(defaultValue = "anonymous") String name) {
        return name;
    }
```

```
http://localhost:8080/api/v1/person?name=abc
----
abc
```
## required false
```java
    @RequestMapping("/person")
    public String hello(@RequestParam(defaultValue = "anonymous") String name, @RequestParam(required = false) String id) {
        return name + " " + id;
    }
```
<!--more-->
```
http://localhost:8080/api/v1/person?name=abc
----
anonymous null
```

# PathVariable

```java
    @RequestMapping("/person/{id}")
    public String getFooById(@PathVariable(required = false) String id) {
        return "ID: " + id;
    }
```

```
http://localhost:8080/person/abc
----
ID: abc
```

# RequestMapping

## headers
```java
    @RequestMapping(value = "/ex/foos", headers = "userid=1", method = RequestMethod.GET)
    public String getFoosWithHeader() {
        return "Get some Foos with Header";
    }
```

```
curl -i -H "userid:1" http://localhost:8080/api/v1/ex/foos
```

## produces
```java
@GetMapping(value = "foos/duplicate", produces = MediaType.APPLICATION_XML_VALUE)
public String duplicate() {
    return "Duplicate";
}
```