---
title: lombok의 @Builder를 추가했더니 잭슨에서 에러가?
categories:
- 트러블슈팅
tags:
- [java, spring, lombok, jackson]
---

![](https://images.unsplash.com/photo-1613834927301-1c96a302e074?ixlib=rb-1.2.1&q=80&cs=tinysrgb&fm=jpg&crop=entropy)

## 문제 발생

한가로운 오후 개발환경에 api 응답 에러가 난다고 문의가 들어왔습니다.  
ES에 에러 로그를 확인해보니 이러한 에러 로그가.. 줄줄..  
(실제 환경을 포스팅할 수 없어 예제 코드를 작성하여 에러를 유도함)  

```bash
Resolved [org.springframework.http.converter.HttpMessageNotReadableException: JSON parse error: Cannot construct instance of `com.example.lomboktest.SampleDto` (although at least one Creator exists): cannot deserialize from Object value (no delegate- or property-based Creator); 
nested exception is com.fasterxml.jackson.databind.exc.MismatchedInputException: Cannot construct instance of `com.example.lomboktest.SampleDto` (although at least one Creator exists): cannot deserialize from Object value (no delegate- or property-based Creator)<EOL> at [Source: (org.springframework.util.StreamUtils$NonClosingInputStream); line: 2, column: 5]]
```

문제가 발생한 dto를 확인해봅시다.  

```java
@Getter
public class SampleDto {
    @Builder.Default
    private String test = "tesT";

    @Builder
    public SampleDto(String test) {
        this.test = test;
    }
}
```

음.. 생성자를 선언하고 빌더를 선언하였는데 무엇이 문제일까?  
에러 로그를 자세히 확인해봅시다.  

```java
JSON parse error:
Cannot construct instance of `com.example.lomboktest.SampleDto` 
(although at least one Creator exists): 
cannot deserialize from Object value (no delegate- or property-based Creator);
```

직역하자면,  JSON 파싱 에러 SampleDto를 생성할 수 없다. (최소 한개의 생성자가 존재하지만)  
객체값으로부터 역직렬화를 할 수 없다. (delegate 또는 property 기반의 생성자가 없음)  

**~~사실 딱 보자마자 알아봤다. 하지만 지식은 output을 해야 오래 기억에 남는법.. 그래서 정리하며 포스팅한다.~~**

## 생성자가 존재하는데, 왜?

눈치채신분들은 아시겠지만, spring은 기본적으로 직렬화/역직렬화에 잭슨 라이브러리를 기본으로 사용하고 있고 그 잭슨 라이브러리는 기본생성자가 없으면 동작하지 않습니다.  

자, 에러 메세지를 출력한 클래스를 추적해봅시다.  

```java
protected Object deserializeFromObjectUsingNonDefault(JsonParser p,
            DeserializationContext ctxt) throws IOException
{
    final JsonDeserializer<Object> delegateDeser = _delegateDeserializer();
    if (delegateDeser != null) {
        final Object bean = _valueInstantiator.createUsingDelegate(ctxt,
                delegateDeser.deserialize(p, ctxt));
        if (_injectables != null) {
            injectValues(ctxt, bean);
        }
        return bean;
    }
    if (_propertyBasedCreator != null) {
        return _deserializeUsingPropertyBased(p, ctxt);
    }
    // 25-Jan-2017, tatu: We do not actually support use of Creators for non-static
    //   inner classes -- with one and only one exception; that of default constructor!
    //   -- so let's indicate it
    Class<?> raw = _beanType.getRawClass();
    if (ClassUtil.isNonStaticInnerClass(raw)) {
        return ctxt.handleMissingInstantiator(raw, null, p, 
"non-static inner classes like this can only by instantiated using default, no-argument constructor");
    }
    return ctxt.handleMissingInstantiator(raw, getValueInstantiator(), p, 
"cannot deserialize from Object value (no delegate- or property-based Creator)");
}
```

에러로그를 전달하는 메서드 이름이 `deserializeFromObjectUsingNonDefault` 요놈이다.  
이름부터 심상치않다.  

좀더 찾아보자.

```java
/*
/**********************************************************
/* Public API implementation; instantiation from JSON Object
/**********************************************************
 */

@Override
public Object createUsingDefault(DeserializationContext ctxt) throws IOException
{
    if (_defaultCreator == null) { // sanity-check; caller should check
        return super.createUsingDefault(ctxt);
    }
    try {
        return _defaultCreator.call();
    } catch (Exception e) { // 19-Apr-2017, tatu: Let's not catch Errors, just Exceptions
        return ctxt.handleInstantiationProblem(_valueClass, null, rewrapCtorProblem(ctxt, e));
    }
}
```

메서드를 꾸준히 따라가다보니 문제 발생 근원지를 찾을 수 있었다.  

> if (_defaultCreator == null) { // sanity-check; caller should check

이 부분이 바로 기본생성자가 null 인지 확인하고 null이면 `super.createUsingDefault(ctxt);` 메서드가 호출될 것이고 에러로그가 출력된 곳으로 바인딩되어 실제 에러로그가 출력될 것입니다.  

반대로 기본생성자가 있다면 호출되는 `_defaultCreator.call();` 를 들어가보면 기본생성자를 사용하는지 살펴보면 확실히 알 수 있을 것 같네요.  

```java
@Override
public final Object call() throws Exception {
    // 31-Mar-2021, tatu: Note! This is faster than calling without arguments
    //   because JDK in its wisdom would otherwise allocate `new Object[0]` to pass
    return _constructor.newInstance((Object[]) null);
}
```

## 그렇다면 에러로그에 있던 no delegate- or property-based 는 뭐지?

우리가 지나쳐온 코드에 답이 존재합니다.  
위의 코드 중 에러메세지 출력된 메서드를 확인해볼까요?  

```java
protected Object deserializeFromObjectUsingNonDefault(JsonParser p,
            DeserializationContext ctxt) throws IOException
{
    final JsonDeserializer<Object> delegateDeser = _delegateDeserializer();
    if (delegateDeser != null) {
        final Object bean = _valueInstantiator.createUsingDelegate(ctxt,
                delegateDeser.deserialize(p, ctxt));
        if (_injectables != null) {
            injectValues(ctxt, bean);
        }
        return bean;
    }
    if (_propertyBasedCreator != null) {
        return _deserializeUsingPropertyBased(p, ctxt);
    }
    // 25-Jan-2017, tatu: We do not actually support use of Creators for non-static
    //   inner classes -- with one and only one exception; that of default constructor!
    //   -- so let's indicate it
    Class<?> raw = _beanType.getRawClass();
    if (ClassUtil.isNonStaticInnerClass(raw)) {
        return ctxt.handleMissingInstantiator(raw, null, p, 
"non-static inner classes like this can only by instantiated using default, no-argument constructor");
    }
    return ctxt.handleMissingInstantiator(raw, getValueInstantiator(), p, 
"cannot deserialize from Object value (no delegate- or property-based Creator)");
}
```

`deserializeFromObjectUsingNonDefault` 메서드명을 살펴보면 기본생성자 없이 역직렬화하는 메서드입니다.  

이 메서드가 탔다면 바로 에러를 찍어야하는데 세부 로직을 보면  

```java
final JsonDeserializer<Object> delegateDeser = _delegateDeserializer();
if (delegateDeser != null) {
    final Object bean = _valueInstantiator.createUsingDelegate(ctxt,
            delegateDeser.deserialize(p, ctxt));
    if (_injectables != null) {
        injectValues(ctxt, bean);
    }
    return bean;
}
```

`delegateDeser != null` delegate가 존재한다면 `createUsingDelegate` 메서드를 통해 객체를 생성하는 로직이 들어가 있습니다.  
지금 저의 상황은 `delegateDeser == null` 이기 때문에 마지막 구문의 에러로그가 출력된 것을 볼 수 있습니다.  

또한 그 밑의 로직인  

```java
if (_propertyBasedCreator != null) {
    return _deserializeUsingPropertyBased(p, ctxt);
}
```

`_propertyBasedCreator` 도 아니기 때문에 저런 에러가 찍힌것입니다.  

## 정리

결국, 저의 문제는 Controller에서 API 요청 스펙을 전달하는 DTO의 객체로써 기본생성자가 없었기에 나타난 문제였습니다.  
간단히 기본생성자를 추가해서 에러를 해결할 수 있었습니다.  
롬복의 @builder는 기본생성자를 생성해주지 않기에 이런 상황이 발생하였습니다.. 😱  

우리 모두 하나만 기억합시다.  
잭슨의 object mapper는 기본생성자를 필요로한다! (기본생성자는 `private` 이어도 된다! **리플렉션**)  
번외로 `@JsonProperty`, `@JsonAutoDetect` 등을 사용하는 프로퍼티 기반 객체 or 생성자를 위임한 경우라면 필요하지 않다!  

끝.
