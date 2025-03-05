# selfBean_java

### Problem
```
현대카드 기존 프레임워크를 프레임워크 플러스로 버전 업그레이드하면서, 
Spring Annotation 기반의 AOP 호출 문제 발생. 
로그 분석 결과, 내부 메서드 호출 시 프록시가 적용되지 않아 
bean이 생성되지 않았다는 메시지와 함께 Bean 호출 에러가 발생. 
이는 AOP 프록시를 거치지 않고 직접 호출되면서 발생한 문제로, 
`Self-Bean` 주입을 통해 해결함.
```

### Definitions
#### BEAN(빈)
```java
Spring 컨테이너가 관리하는 객체로, DI(Dependency Injection)를 통해 
애플리케이션 전반에서 재사용 가능함.
@Component, @Service, @Repository, @Bean 등의 어노테이션을 사용해 정의할 수 있음.

- 예제
@Service
public class PaymentService {
    public void processPayment() {
        System.out.println("Processing payment...");
    }
}
```

#### AOP (Aspect-Oriented Programming, 관점 지향 프로그래밍)
```java
핵심 로직과 공통 관심 사항(트랜잭션, 로깅, 보안 등)을 
분리하여 코드의 중복을 줄이는 프로그래밍 패러다임.
Spring에서는 프록시(Proxy) 객체를 사용해 
메서드 호출을 가로채고(@Transactional, @Async 등), 추가적인 기능을 실행함.

- 예제 (@Transactional 사용)
@Service
public class OrderService {
    
    @Transactional  // AOP 적용 (트랜잭션 관리)
    public void createOrder() {
        System.out.println("Creating order...");
    }
}
```

#### Annotation(어노테이션)
```java
Java 코드에 메타데이터를 추가하여 런타임 또는 
컴파일 타임에 특정 동작을 수행하도록 지시하는 마커.
Spring에서는 빈 등록, AOP 적용, 의존성 주입 등에 활용됨.

- 예제
@Component  // Bean 등록
public class MyComponent {
}

@Autowired  // 의존성 주입
private MyComponent myComponent;
```

### Causes
오류가 발생한 원인은 3가지 정도로 분류 가능
- 
(1) Spring AOP가 프록시를 통해 동작하는 방식 변화
```
Spring에서 AOP를 사용하면, 어노테이션이 붙은 메서드는 프록시 객체(proxy object) 를 통해 호출
기존버전에서는 클래스 내부에서 Bean을 호출할 때에도 AOP 프록시가 개입할 수 있었지만
버전 업 이후, 바이트코드 최적화 및 프록시 호출 방식이 변경되면서 내부 호출이 
프록시를 거치지 않고 직접 호출(내부 참조)이 될 수 있음.
따라서 직접 호출하면 프록시를 거치지 않아 
AOP 기능(트랜잭션, 비동기 실행 등)이 정상적으로 동작하지 않음.
```

(2) JIT(Just-In-Time) 컴파일러 및 인라이닝 최적화 변화
```
기존 버전에서 JVM의 JIT(Just-In-Time) 컴파일러와 C2 최적화 기법이 변경됨.
Java 17 이후, 메서드 인라이닝(Inline Optimization) 이 더욱 적극적으로 수행되면서 
내부 호출이 프록시 객체를 경유하지 않고 직접 호출로 변환될 가능성이 증가함.
기존에는 호출이 프록시를 거쳐 실행되었지만, 
최적화된 Java 버전에서는 직접 호출되면서 AOP 기능이 우회되는 문제 발생.
```

(3) CGLIB vs JDK Dynamic Proxy 차이
```
Spring의 기본 프록시 방식은 
JDK Dynamic Proxy(인터페이스 기반) 또는 CGLIB(클래스 기반)인데,
버전업 이후 CGLIB의 내부 구현 방식이 최적화되면서 
내부 호출이 프록시를 우회할 가능성 증가.
클래스 내부 참조가 직접 호출로 변환되면서 AOP 기능이 적용되지 않는 경우가 있음.
```

### Solution
```java
실무에서의 사례는 1번의 경우로 판단됨.
SpringAOP가 프록시를 통해 동작하는 방식이 변화하면서
Annotation이 붙은 메서드가 직접 호출하면 프록시를 거치않아
AOP 기능이 정상적으로 동작하지 않음.

위의 Case를 Self-Bean호출을 통해 해결함.
```

#### As-Is
```java
@BXMApplication
public class Test {
    private DataBase dataBase;

    public void process() {
        if (dataBase != null) {
            dataBase.application();  // 문제 발생: AOP 미적용 가능성
        }
    }
}

/* dataBase.application() 호출 시, 프록시를 거치지 않고 직접 호출됨.
AOP가 적용된 메서드(@Transactional, @Async 등)라면 Spring의 프록시 기능이 작동하지 않음.
프레임워크 플러스 버전에서 
내부 호출 방식이 최적화되면서 프록시가 무시될 가능성이 증가. */
````

#### To-Be
```java
@BXMApplication
public class Test {

    @Autowired
    private Test self;  // Self-Bean 주입
    private DataBase dataBase;

    public void process() {
        if (dataBase != null) {
            self.callDatabase();  // 프록시를 통해 호출
        }
    }

    @Transactional
    public void callDatabase() {
        dataBase.application();  // AOP 적용됨
    }
}

/* 자기 자신을 Spring 컨테이너에서 주입
내부 메서드 호출을 프록시 객체를 통해 수행 (self.callDatabase();)
AOP(@Transactional, @Async 등) 기능이 정상적으로 동작
callDatabase()는 AOP 프록시를 거쳐 호출되므로 
트랜잭션 및 비동기 실행이 정상 동작함. */

```

#### 정리
| 구분 | AS-IS (기존 방식) | TO-BE (Self-Bean 적용) |
|------|----------------|----------------|
| **내부 호출 방식** | `this.method()` 사용 | `self.method()` 사용 |
| **AOP 적용 여부** | 미적용 (프록시 무시됨) | 적용됨 (프록시 호출) |
| **문제점** | 트랜잭션/비동기 미적용 | 트랜잭션/비동기 정상 작동 |
| **해결책** | Self-Bean 주입 필요 | `@Autowired private Test self;` 적용 |


