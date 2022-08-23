---
title: 이펙티브 자바 톺아보기 - 아이템 3. 생성자나 열거 타입으로 싱글턴임을 보증하라
categories:
- 이펙티브 자바 톺아보기 시리즈
tags:
- java
---

![](/assets/images/posts/effective-java/main.png)

## 핵심 정리

### private 생성자 + public static final 필드를 사용하는 방법

#### 장점 1. 간결하고 싱글턴임을 API에 들어낼 수 있다

```java
public class Elvis {
  
    /**  
     * 싱글톤 오브젝트  
     */  
    public static final Elvis INSTANCE = new Elvis();  
    private Elvis(){}
}
```

#### 단점 1. 싱글톤을 사용하는 클라이언트를 테스트하기 어려워진다  

- 싱글톤을 사용하는 클라이언트 객체를 테스트하기 위해 싱글톤 객체를 항상 호출해야한다.
- 테스트 하기위한 mock 객체를 주입하기 어려움

```java
// 싱글톤 객체를 사용하는 클라이언트 객체
public class Concert {  
    private boolean lightsOn;  
    private boolean mainStateOpen;  
    private Elvis elvis;
  
    public Concert(Elvis elvis) {  
        this.elvis = elvis;  
    }  
  
    public void perform() {  
        mainStateOpen = true;  
        lightsOn = true;  
        elvis.sing(); // 싱글톤 객체
    }
}
```

#### 단점 2. 리플렉션으로 private 생성자를 호출할 수 있다  

```java
public class Elvis implements Serializable {  
  
    /**  
     * 싱글톤 오브젝트  
     */  
    public static final Elvis INSTANCE = new Elvis();  
    private static boolean created; // 인스턴스 생성 flag 값 체크
  
    private Elvis() {  
        if (created) {  
            throw new UnsupportedOperationException("can't be created by constructor.");  
        }  
  
        created = true;  
    }
}
```

#### 단점 3. 역직렬화 할 때 새로운 인스턴스가 생길 수 있다  

```java
// 역직렬화로 여러 인스턴스 만들기  
public class ElvisSerialization {   
    public static void main(String[] args) {  
        try (ObjectOutput out = new ObjectOutputStream(new FileOutputStream("elvis.obj"))) {  
            out.writeObject(Elvis.INSTANCE);  
        } catch (IOException e) {  
            e.printStackTrace();  
        }  
  
        try (ObjectInput in = new ObjectInputStream(new FileInputStream("elvis.obj"))) {  
            Elvis elvis3 = (Elvis) in.readObject();  
            System.out.println(elvis3 == Elvis.INSTANCE); // False -> readResolve() 선언시 True
        } catch (IOException | ClassNotFoundException e) {  
            e.printStackTrace();  
        }  
    }  
  
}

public class Elvis implements Serializable {  
  
    /**  
     * 싱글톤 오브젝트  
     */  
    public static final Elvis INSTANCE = new Elvis();  
    private static boolean created;  
  
    private Elvis() {  
    }

    // 역직렬화시 새로운 인스턴스를 생성하지 않고 readResolve() 메서드를 실행함
    private Object readResolve() {
        return INSTANCE;  
    }
}  
```

### private 생성자 + 정적 팩터리 메서드를 사용하는 방법

#### 장점 1. API를 바꾸지 않고도 싱글턴이 아니게 변경할 수 있다  

#### 장점 2. 정적 팩터리를 제네릭 싱글턴 팩터리로 만들 수 있다  

#### 장점 3. 정적 팩터리의 메서드 참조를 공급자(Supplier)로 사용할 수 있다

#### 단점은 public static final 필드를 사용하는 방법과 동일하다
