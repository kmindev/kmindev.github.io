---
title: "[Spring] HttpServletRequest 스트림 소진으로 인한 캐시 적용"
date: 2026-08-30 00:00:00 +0900
categories: [Language/Programming]
tags: [Spring, HttpServletRequest, InputStream, Filter, 트러블슈팅]
---

모든 요청에 대해 로깅하기 위해 보통 필터/인터셉터를 많이들 활용한다.

하지만, 필터/인터셉터 레벨에서 Request body를 무작정 로깅을 할 경우, 문제가 발생한다.

어떤 문제가 발생하는 지와 왜 발생하는 지, 어떻게 해결할 수 있는 지 정리해보자.

## 1. 문제


### 문제가 되는 코드

```java
@Slf4j
@Component
public class RequestBodyLoggingFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;

        // getInputStream()으로 body를 끝까지 읽어버림 -> stream이 소진된다.
        String body = StreamUtils.copyToString(httpRequest.getInputStream(), StandardCharsets.UTF_8);
        log.info("[Filter] Request Body: {}", body);

        // 소진된 원본 request를 그대로 다음 체인(컨트롤러)에 전달
        chain.doFilter(request, response);
    }
}

@PostMapping("/test")
public String test(@RequestBody TestRequest request) {
    log.info("[Controller] Request Body: {}", request);
    return "OK";
}
```

### 호출 결과 
예외 메시지(`Required request body is missing`)를 보면 request body가 필수인데, 찾을 수 없다는 것이다.

```
com.test.RequestBodyLoggingFilter        : [Filter] Request Body: {
  "message": "test"
}
w.s.m.s.DefaultHandlerExceptionResolver : Resolved [org.springframework.http.converter.HttpMessageNotReadableException: Required request body is missing: public java.lang.String com.test.TestController.test(com.test.TestController$TestRequest)]
```



## 2. 발생 원인

필터에서 `request.getInputStream()`read() 하는 순간 stream이 소진된다. (`StreamUtils.copyToString()` 내부적으로 read() 호출함)

Controller 메서드 파라미터에 `@RequestBody`가 붙어있으면 `RequestResponseBodyMethodProcessor`(`HandlerMethodArgumentResolver`의 구현체)가 바인딩을 담당하는데, 
이 리졸버도 내부적으로 stream을 읽는다. 

필터에서 이미 끝까지 읽어버린 stream이기 때문에, 여기서는 빈 stream만 남아있게 된다.

정리하면 다음과 같다.

1. 필터에서 stream `read()` → stream 소진
2. `RequestResponseBodyMethodProcessor.resolveArgument()`가 내부적으로 stream을 다시 읽으려 하지만, 이미 소진된 상태라 `null` 반환
3. `@RequestBody`는 기본적으로 `required = true`이므로, `null`인 채로 `HttpMessageNotReadableException` 예외 발생

```java
// RequestResponseBodyMethodProcessor.resolveArgument()
@Override
public @Nullable Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
    parameter = parameter.nestedIfOptional();
    Object arg = readWithMessageConverters(webRequest, parameter, parameter.getNestedGenericParameterType());
    // .. 생략
}

@Override
protected @Nullable Object readWithMessageConverters(NativeWebRequest webRequest, MethodParameter parameter, Type paramType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {
    ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
    Object arg = readWithMessageConverters(inputMessage, parameter, paramType); // null (내부에서 stream을 읽지만 이미 소진되어 null 반환)
    if (arg == null && checkRequired(parameter)) { // true (@RequestBody는 기본적으로 required = true이므로 null인 경우 예외 발생)
        throw new HttpMessageNotReadableException("Required request body is missing: " +
            parameter.getExecutable().toGenericString(), inputMessage);
    }
    return arg;
}
```

![resolve-argument-null-debugger.png](/assets/img/posts/resolve-argument-null-debugger.png)

## 3. 해결 방법

해결 방법은 간단하다.

stream은 소진되기 때문에 캐시를 해주면 된다.

두 가지 방법을 다뤄보겠다.

### 3-1. 프레임워크에서 제공해주는 ContentCachingRequestWrapper 활용하는 방법
- Spring에서 제공해주는 `ContentCachingRequestWrapper`를 활용할 수 있다.
```java
@Slf4j
@Component
public class RequestBodyLoggingFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
        throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        ContentCachingRequestWrapper wrappedRequest = new ContentCachingRequestWrapper(httpRequest, 4096);
        String body = new String(wrappedRequest.getContentAsByteArray(), wrappedRequest.getCharacterEncoding());
        log.info("[Filter] Request Body: {}", body);
        
        chain.doFilter(wrappedRequest, response);
    }
}
```

```
com.test.RequestBodyLoggingFilter        : [Filter] Request Body: 
com.test.TestController                  : [Controller] Request Body: TestRequest[message=test]
```

결과를 보면 필터에서 Request body가 비어있는 것을 확인할 수 있다.

이는 `ContentCachingRequestWrapper`가 stream 자체를 캐싱하는 게 아니라, 실제로 read된 만큼만 content로 캐싱하기 때문이다.

그러므로 `doFilter()` 전(stream을 읽기 전)에 `wrappedRequest.getContentAsByteArray()`를 호출해도 빈 값이 나오는 것이었다.

![content-caching-input-stream-read.png](/assets/img/posts/content-caching-input-stream-read.png)

doFilter 이후에 `wrappedRequest.getContentAsByteArray()`를 호출해보자.

resolver에서 stream을 read 하기 때문에 content가 캐시되어 있을 것이다!

```java
@Slf4j
@Component
public class RequestBodyLoggingFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
        throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        ContentCachingRequestWrapper wrappedRequest = new ContentCachingRequestWrapper(httpRequest, 4096);
        
        chain.doFilter(wrappedRequest, response);

        String body = new String(wrappedRequest.getContentAsByteArray(), wrappedRequest.getCharacterEncoding());
        log.info("[Filter] Request Body: {}", body);
    }
}
```

#### 결과 (성공!)
```
com.test.TestController                  : [Controller] Request Body: TestRequest[message=test]
com.test.RequestBodyLoggingFilter        : [Filter] Request Body: {
  "message": "test"
}
```

하지만 위 방법은 `doFilter()` 이후에 로깅하도록 순서가 강제된다는 단점이 있다.

즉, 필터 -> 컨트롤러 -> 비즈니스 로직 처리 후에야 로깅이 이뤄진다는 것이다.

예외가 발생하는 경우 요청 로깅이 누락되는 등의 문제가 발생할 수 있다.

그래서 직접 구현하는 방법이 더 실용적일 수 있다.

### 3-2. 커스텀 구현하는 방법

생성자 시점에 stream을 미리 다 읽어 body를 캐싱해두고, `getInputStream()`을 호출할 때마다 캐싱해둔 body로 새 stream을 만들어 반환하는 방식으로 구현할 수 있다.

```java
public class CachedBodyHttpServletRequest extends HttpServletRequestWrapper {
    private final byte[] cachedBody;

    public CachedBodyHttpServletRequest(HttpServletRequest request) throws IOException {
        super(request);
        this.cachedBody = StreamUtils.copyToByteArray(request.getInputStream()); // 생성자 시점에 원본 stream을 즉시 끝까지 읽어 byte[]로 캐싱해둔다.
    }

    @Override
    public ServletInputStream getInputStream() {
        ByteArrayInputStream buffer = new ByteArrayInputStream(cachedBody); // 호출할 때마다 캐싱해둔 byte[]로 새 stream을 만들어 반환 -> 몇 번이든 처음부터 다시 읽을 수 있다.
        return new ServletInputStream() {
            @Override
            public int read() {
                return buffer.read();
            }

            @Override
            public boolean isFinished() {
                return buffer.available() == 0;
            }

            @Override
            public boolean isReady() {
                return true;
            }

            @Override
            public void setReadListener(jakarta.servlet.ReadListener readListener) {
                throw new UnsupportedOperationException();
            }
        };
    }

    public byte[] getBody() {
        return cachedBody;
    }
}

@Slf4j
@Component
public class RequestBodyLoggingFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
        throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;

        // 생성자에서 이미 body 전체를 byte[]로 캐싱해둔 상태 -> getBody()로 바로 조회
        CachedBodyHttpServletRequest cachedRequest = new CachedBodyHttpServletRequest(httpRequest);

        log.info("[Filter] Request Body: {}", new String(cachedRequest.getBody(), StandardCharsets.UTF_8));

        chain.doFilter(cachedRequest, response);
    }
}
```

#### 결과 (성공!)
```
com.test.RequestBodyLoggingFilter        : [Filter] Request Body: {
  "message": "test"
}
com.test.TestController                  : [Controller] Request Body: TestRequest[message=test]
```

## 4. 정리

stream은 한 번 읽으면 소진되므로, 캐시를 해둬야 한다.

Spring에서 제공하는 `ContentCachingRequestWrapper`를 활용할 수 있지만, stream 자체를 캐싱하는 게 아니라 실제로 읽힌 만큼만 캐싱하는 방식이라 사용에 제한이 있다.

자유롭게 사용하고 싶다면 커스텀 구현 방식이 낫다.

다만 메모리 사용량이 늘어나므로, 의미 있는 데이터에만 캐싱을 적용해서 로깅해야 한다.

동영상 같은 바이너리 데이터는 로깅에 의미도 없고 용량도 커서 메모리를 과하게 사용하게 되므로, 이런 데이터는 캐싱 대상에서 제외하는 게 좋다.

