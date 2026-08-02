---
title: "String을 조립해서 전달하는 방식 vs log {} (파라미터를 넘기는 방식)"
date: 2026-08-02 00:00:00 +0900
categories: [Java]
tags: [Java, Logging, SLF4J]
---

로그를 찍을 때 단순하게 `String`을 조립해서 전달하는 것과 `{}`를 통해 파라미터를 넘기는 방식이 있다.

이 두 방식은 String 생성 시점에서 차이가 있다고 알고 있고, 일반적으로 파라미터를 통해 넘기는 방식을 사용하라고 한다.

실제로 어떻게 다른 지 코드를 통해 직접 확인해보자.

## 예제 코드

- `print()`를 통해 `toString()` 호출 여부를 확인하려고 한다.

```java
@AllArgsConstructor
public class User {
    String id, name;

    @Override
    public String toString() {
        System.out.println("toString() 호출");
        return id + ":" + name;
    }
}
```

- 로그레벨은 `INFO` 설정했다.

```yaml
logging:
  level:
    root: INFO
```

## String을 조립해서 전달하는 방식

```java
public void test() {
    User user = new User("123", "홍길동");
    log.debug("User: " + user);  // + 연산자로 문자열 조립
}
```

### 결과

- 로그레벨이 INFO로 설정했기에, DEBUG 레벨의 로그는 출력되지 않는다. 
- 로그 출력과 상관없이 `toString()`은 항상 호출된다.

![toString()이 항상 호출되는 결과](/assets/img/posts/string-concat-tostring-called.png)

### 이유
- `+` 는 Java의 연산자이므로, `String + obj`는 `obj.toString()`을 즉시 호출해 String을 만든다.
- 즉, 로깅 라이브러리가 레벨을 확인할 기회조차 없이 `msg(String)`을 만들어서 전달한다.
```java
// org.slf4j.helpers.AbstractLogger
public void debug(String msg) {
    if (isDebugEnabled()) {
        handle_0ArgsCall(Level.DEBUG, null, msg, null);
    }
}
```

## log {} (파라미터 전달) 방식

```java
public void test() {
    User user = new User("123", "홍길동");
    log.debug("User: {}", user); 
}
```

### 결과
- 로그레벨이 `INFO`로 설정했기에, `DEBUG` 레벨의 로그는 출력되지 않는다.
- `toString()` 자체도 호출되지 않는다.

### 이유
- `{}` 방식은 `user` 객체의 **참조값**만 메서드에 전달한다.
- `toString()`은 레벨을 확인한 뒤, 실제로 로그를 남길 필요가 있을 때만 호출한다.

```java
// org.slf4j.helpers.AbstractLogger
public void debug(String format, Object arg) {
    if (isDebugEnabled()) {
        handle_1ArgsCall(Level.DEBUG, null, format, arg);
    }
}
```

## 결론

`{}` 파라미터 방식을 사용하면 String 생성 자체를 레벨이 활성화된 경우에만 수행하므로, 불필요한 메모리 낭비를 막을 수 있다.
