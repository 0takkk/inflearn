# 스프링 핵심 원리 - 고급편

## 쓰레드 로컬 - ThreadLocal

스프링에서 싱글톤으로 객체를 관리한다.  
따라서, `FieldLogTrace`를 싱글톤으로 관리한다.  
싱글톤 객체를 사용할 때 동시성 이슈를 조심해야한다.  
`http://localhost:8080/v3/request?itemId=hello` 를 1초 이내에 여러번 실행했을 경우,

```
2023-08-16 14:53:05.799  INFO 75512 --- [nio-8080-exec-1] h.advanced.trace.logtrace.FieldLogTrace  : [dafc3e56] OrderController.request()
2023-08-16 14:53:05.801  INFO 75512 --- [nio-8080-exec-1] h.advanced.trace.logtrace.FieldLogTrace  : [dafc3e56] |-->OrderService.orderItem()
2023-08-16 14:53:05.802  INFO 75512 --- [nio-8080-exec-1] h.advanced.trace.logtrace.FieldLogTrace  : [dafc3e56] |   |-->OrderRepository.save()
2023-08-16 14:53:05.835  INFO 75512 --- [nio-8080-exec-2] h.advanced.trace.logtrace.FieldLogTrace  : [dafc3e56] |   |   |-->OrderController.request()
2023-08-16 14:53:05.835  INFO 75512 --- [nio-8080-exec-2] h.advanced.trace.logtrace.FieldLogTrace  : [dafc3e56] |   |   |   |-->OrderService.orderItem()
2023-08-16 14:53:05.835  INFO 75512 --- [nio-8080-exec-2] h.advanced.trace.logtrace.FieldLogTrace  : [dafc3e56] |   |   |   |   |-->OrderRepository.save()
2023-08-16 14:53:06.805  INFO 75512 --- [nio-8080-exec-1] h.advanced.trace.logtrace.FieldLogTrace  : [dafc3e56] |   |<--OrderRepository.save() time=1003ms
2023-08-16 14:53:06.806  INFO 75512 --- [nio-8080-exec-1] h.advanced.trace.logtrace.FieldLogTrace  : [dafc3e56] |<--OrderService.orderItem() time=1005ms
2023-08-16 14:53:06.806  INFO 75512 --- [nio-8080-exec-1] h.advanced.trace.logtrace.FieldLogTrace  : [dafc3e56] OrderController.request() time=1007ms
2023-08-16 14:53:06.838  INFO 75512 --- [nio-8080-exec-2] h.advanced.trace.logtrace.FieldLogTrace  : [dafc3e56] |   |   |   |   |<--OrderRepository.save() time=1003ms
2023-08-16 14:53:06.840  INFO 75512 --- [nio-8080-exec-2] h.advanced.trace.logtrace.FieldLogTrace  : [dafc3e56] |   |   |   |<--OrderService.orderItem() time=1005ms
2023-08-16 14:53:06.840  INFO 75512 --- [nio-8080-exec-2] h.advanced.trace.logtrace.FieldLogTrace  : [dafc3e56] |   |   |<--OrderController.request() time=1005ms
```

위와 같이 예상과 다르게 출력된다.  
`[nio-8080-exec-1]`, `[nio-8080-exec-2]` 스레드 2개가 같은 객체를 사용해서 동시성 이슈가 발생한 것이다.
<br>

`ThreadLocal`을 통해 해결할 수 있다.  
스레드 로컬을 사용하면 각 스레드마다 별도의 내부 저장소를 제공한다.  
따라서, 같은 인스턴스의 스레드 로컬 필드에 접근해도 문제 없다.

```
2023-08-16 23:08:30.166  INFO 82257 --- [nio-8080-exec-3] h.a.trace.logtrace.ThreadLocalLogTrace   : [5d8e0d9d] OrderController.request()
2023-08-16 23:08:30.167  INFO 82257 --- [nio-8080-exec-3] h.a.trace.logtrace.ThreadLocalLogTrace   : [5d8e0d9d] |-->OrderService.orderItem()
2023-08-16 23:08:30.167  INFO 82257 --- [nio-8080-exec-3] h.a.trace.logtrace.ThreadLocalLogTrace   : [5d8e0d9d] |   |-->OrderRepository.save()
2023-08-16 23:08:30.365  INFO 82257 --- [nio-8080-exec-4] h.a.trace.logtrace.ThreadLocalLogTrace   : [0ec5e389] OrderController.request()
2023-08-16 23:08:30.366  INFO 82257 --- [nio-8080-exec-4] h.a.trace.logtrace.ThreadLocalLogTrace   : [0ec5e389] |-->OrderService.orderItem()
2023-08-16 23:08:30.366  INFO 82257 --- [nio-8080-exec-4] h.a.trace.logtrace.ThreadLocalLogTrace   : [0ec5e389] |   |-->OrderRepository.save()
2023-08-16 23:08:31.173  INFO 82257 --- [nio-8080-exec-3] h.a.trace.logtrace.ThreadLocalLogTrace   : [5d8e0d9d] |   |<--OrderRepository.save() time=1005ms
2023-08-16 23:08:31.174  INFO 82257 --- [nio-8080-exec-3] h.a.trace.logtrace.ThreadLocalLogTrace   : [5d8e0d9d] |<--OrderService.orderItem() time=1007ms
2023-08-16 23:08:31.175  INFO 82257 --- [nio-8080-exec-3] h.a.trace.logtrace.ThreadLocalLogTrace   : [5d8e0d9d] OrderController.request() time=1009ms
2023-08-16 23:08:31.369  INFO 82257 --- [nio-8080-exec-4] h.a.trace.logtrace.ThreadLocalLogTrace   : [0ec5e389] |   |<--OrderRepository.save() time=1003ms
2023-08-16 23:08:31.370  INFO 82257 --- [nio-8080-exec-4] h.a.trace.logtrace.ThreadLocalLogTrace   : [0ec5e389] |<--OrderService.orderItem() time=1004ms
2023-08-16 23:08:31.371  INFO 82257 --- [nio-8080-exec-4] h.a.trace.logtrace.ThreadLocalLogTrace   : [0ec5e389] OrderController.request() time=1006ms
```

<br>

`ThreadLocal`의 값을 사용한 후 제거하지 않으면 WAS처럼 스레드 풀을 사용하는 경우에 심각한 문제가 발생할 수 있다.  
스레드 풀에서는 스레드를 재사용하기 때문에 이전에 사용한 데이터가 살아있게 되고, 다른 사용자가 해당 데이터를 사용할 수 있다.  
따라서, `ThreadLocal.remove()`로 스레드 로컬를 제거해야 한다.  
<br>

## 템플릿 메서드 패턴과 콜백 패턴

### 템플릿 메서드 패턴

요구사항에 따라 로그 추적기를 구현했지만, 핵심 기능보다 부가 기능(로그를 출력하는 코드)이 더 길어졌다.  
`Controller`, `Service`, `Repository`를 보면 로그 추적기를 사용하는 구조는 모두 동일하다.

```
  TraceStatus status = null;
  try {
      status = trace.begin("message");

      //핵심 기능 호출

      trace.end(status);
  } catch (Exception e) {
      trace.exception(status, e);
      throw e;
  }
```

<br>

`템플릿 메서드 패턴`을 사용하면 핵심 기능(변하는) 부분과 부가 기능(변하지 않는, 로그 추적) 부분을 분리하여 개발할 수 있다.

<img width="599" alt="image" src="https://github.com/0takkk/inflearn/assets/89503136/57ed3a91-0391-4353-91be-578068d09bb0">

변하지 않는 부분을 `execute()`에 두고, 변하는 부분을 `call()` 추상 메서드로 만든다.  
`템플릿 메서드 패턴`은 부모 클래스에 변하지 않는 템플릿 코드를 두고, 변하는 부분은 자식 클래스에 두고 상속과 오버라이딩을 통해 처리한다.
<br>

`템플릿 메서드 패턴`은 `상속`을 사용한다.  
자식 클래스는 부모 클래스의 기능을 사용하지 않지만 모든 기능을 상속받고 의존하고 있다.  
이런 잘못된 의존관계로 인해 부모 클래스가 수정되면, 자식 클래스도 영향을 받는다.  
또한, 상속 구조로 인해 별도의 클래스나 익명 내부 클래스를 만들어야 한다.  
이러한 단점을 해결한 디자인 패턴이 `전략 패턴`이다.
<br>

### 전략 패턴

<img width="596" alt="image" src="https://github.com/0takkk/inflearn/assets/89503136/c3cf349b-45b9-4379-bb1d-4baa75c13f1c">

`전략 패턴`은 변하지 않는 부분은 `Context`에 두고, 변하는 부분을 `Strategy`라는 인터페이스를 만들고 구현해서 해결한다.  
따라서, 상속이 아니라 `위임`으로 문제를 해결한다.  
<br>

`Context`가 `Strategy` 인터페이스에만 의존한다.  
덕분에, `Strategy`의 구현체를 변경하거나 새로 만들어도 `Context` 코드에 영향을 주지 않는다.
스프링에서 의존관계 주입에 사용하는 방식이 `전략 패턴`이다.

### 템플릿 콜백 패턴

스프링에서 `ContextV2`와 같이 파라미터로 전달하는 방식의 전략 패턴을 `템플릿 콜백 패턴`이라고 한다.  
`템플릿 콜백 패턴`은 GOF 패턴은 아니고, 스프링 내부에서 자주 사용한다.  
<br>

로그 추적기를 적용하기 위해 다양한 패턴을 사용해봤지만, 원본 코드를 수정해야한다.  
원본 코드를 수정하지 않고 로그 추적기(부가 기능)를 적용하기 위해서는 `프록시`개념을 알아야 한다.

<br>

## 프록시 패턴과 데코레이터 패턴

`프록시`를 사용하면 원본 코드를 수정하지 않고, 부가 기능을 도입할 수 있다.  
<img width="593" alt="image" src="https://github.com/0takkk/inflearn/assets/89503136/ffda1fe8-83c9-485b-9e4c-f851b3f970e4">
<img width="593" alt="image" src="https://github.com/0takkk/inflearn/assets/89503136/913912f1-b159-42cf-806d-7391ae4fe8ee">

`프록시`는 클라이언트가 요청한 결과를 서버에 직접 요청하는 것이 아니라, 대리자를 통해 간접적으로 서버에 요청하는 것이다.

<img width="591" alt="image" src="https://github.com/0takkk/inflearn/assets/89503136/805cb7e3-5ce7-4c27-bbec-9f1b9df551e0">

서버와 프록시는 같은 인터페이스를 사용해야 한다.  
클라이언트가 사용하는 서버 객체를 프록시 객체로 변경해도 클라이언트 코드를 변경하지 않고 동작할 수 있어야 한다.  
<br>

`프록시`는 `접근 제어`와 `부가 기능 추가`의 큰 주요 기능이 있다.  
`접근 제어`

- 권한에 따른 접근 차단
- 캐싱
- 지연 로딩

`부가 기능 추가`

- 원래 서버가 제공하는 기능에 더해서 부가 기능을 수행한다.

> 둘 다 프록시를 사용하지만 GOF 디자인 패턴에서는 이 둘의 의도에 따라 프록시 패턴과 데코레이터 패턴으로 구분한다.
>
> - 프록시 패턴 : 접근 제어가 목적
> - 데코레이터 패턴 : 새로운 기능 추가가 목적

<br>

프록시를 사용해서 기존 코드를 수정하지 않고 부가 기능을 적용할 수 있다.  
하지만, 프록시 클래스를 너무 많이 만들어야 한다.  
이런 경우 `동적 프록시` 기술을 사용할 수 있다.

<br>

## 동적 프록시 기술

### 리플렉션

자바가 기본으로 제공하는 JDK 동적 프록시 기술이나 CGLIB 같은 프록시 생성 오픈소스를 사용하면 프록시 객체를 동적으로 만들어낼 수 있다.  
JDK 동적 프록시를 이해하기 위해서는 먼저 자바 리플렉션 기술을 이해해야 한다.  
<br>

호출하는 메서드 `target.callA()`, `target.callB()` 이 부분만 동적으로 처리하도록 하는 기술이 `리플렉션`이다.

```
// 클래스 정보 획득
Class classHello = Class.forName("hello.proxy.jdkdynamic.ReflectionTest$Hello");

Hello target = new Hello();
// callA 메서드 정보 획득
Method methodCallA = classHello.getMethod("callA");
Object result1 = methodCallA.invoke(target);
log.info("result1 = {}", result1);
```

`getMethod`의 String 값을 동적으로 변경할 수 있다.

```
private void dynamicCall(Method method, Object target) throws Exception{
    log.info("start");
    Object result = method.invoke(target);
    log.info("result = {}", result);
}
```

하지만, `리플렉션` 기술은 런타임에 동작하기 때문에, 컴파일 시점에 오류를 잡을 수 없다.

### JDK 동적 프록시

`JDK 동적 프록시`는 `InvocationHandler` 인터페이스를 구현해서 작성해야 한다.  
또한, 인터페이스가 필수이다.

### CGLIB

바이트 코드를 조작해서 동적으로 클래스를 생성하는 기술.  
인터페이스 없이 구체 클래스만 가지고 동적 프록시 기술을 사용할 수 있다.  
`MethodInterceptor` 인터페이스 구현해야 한다.

<br>

## 스프링이 지원하는 프록시

### 프록시 팩토리

스프링은 동적 프록시를 통합해서 편리하게 만들어주는 `프록시 팩토리` 기능을 제공한다.  
`프록시 팩토리`는 인터페이스가 있으면 JDK 동적 프록시, 구체 클래스만 있으면 CGLIB를 사용한다.  
`InvocationHandler`나 `MethodInterceptor`를 신경쓰지 않고, `Advice`만 만들면 된다.

<img width="804" alt="image" src="https://github.com/0takkk/inflearn/assets/89503136/d97e369a-d206-4cb3-889e-4531de36c472">

### 포인트컷, 어드바이스, 어드바이저

`포인트컷(Pointcut)` : 어디에 부가 기능을 적용할지 필터링 로직. 주로 클래스와 메서드 이름으로 필터링.  
`어드바이스(Advice)` : 이전에 본 것 처럼 프록시가 호출하는 부가 기능. 프록시 로직  
`어드바이저(Advisor)` : 하나의 포인트컷과 하나의 어드바이스를 가지고 있는 것.

<br>

## 빈 후처리기

스프링에서 빈 저장소에 등록할 객체를 조작하고 싶다면 `빈 후처리기`를 사용하면 된다.

> `빈 후처리기` 과정
>
> 1. **생성** : 스프링 빈 대상이 되는 객체를 생성
> 2. **전달** : 생성된 객체를 빈 저장소에 등록하기 직전에 빈 후처리기에 전달
> 3. **후 처리 작업** : 빈 후처리기는 전달된 스프링 빈 객체를 조작하거나 다른 객체로 변경
> 4. **등록** : 빈 후처리기는 빈을 반환

<img width="821" alt="image" src="https://github.com/0takkk/inflearn/assets/89503136/1f9eae66-9f75-4d17-ae7f-ff104f189372">

`빈 후처리기`를 사용해서 실제 객체 대신 `프록시 객체`를 스프링 빈으로 등록할 수 있다.  
스프링 부트는 `AnnotationAwareAspectJAutoProxyCreator`라는 `빈 후처리기`가 자동으로 등록된다.  
따라서, `Advisor`만 빈으로 등록하면 자동으로 프록시 객체를 만들어준다.

<img width="814" alt="image" src="https://github.com/0takkk/inflearn/assets/89503136/b709d129-5fb2-4a44-8247-620a02e9e2e6">

<br>
<br>

## @Aspect AOP

스프링은 `@Aspect` 어노테이션으로 포인트컷과 어드바이스로 구성된 어드바이저를 생성할 수 있다.

```
@Around("execution(* hello.proxy.app..*(..))")
public Object execute(ProceedingJoinPoint joinPoint) throws Throwable {
    TraceStatus status = null;
    try {
        String message = joinPoint.getSignature().toShortString();
        status = trace.begin(message);

        Object result = joinPoint.proceed();

        trace.end(status);
        return result;
    } catch (Exception e) {
        trace.exception(status, e);
        throw e;
    }
}
```

`@Around`가 포인트컷, 메서드 안의 내용이 어드바이스이다.

<br>

## 스프링 AOP 개념

애플리케이션 로직은 크게 `핵심 기능`과 `부가 기능`으로 나뉜다.  
`핵심 기능`은 해당 객체가 제공하는 고유의 기능.  
`부가 기능`은 핵심 기능을 보조하는 기능. ex) 로그 추적, 트랜잭션 등

<br>

`핵심 기능`과 `부가 기능`을 함께 코드로 작성하면, 단일 책임 원칙을 위반하게 된다.  
또한, `부가 기능`은 여러 로직에서 필요로 하는데, 이는 코드의 중복을 발생시키고 유지보수가 어려워 진다.

<br>

`핵심 기능`과 `부가 기능`을 분리해서 개발 및 관리하도록 하는 기술이 `AOP(관심 지향 프로그래밍)`이다.

<br>

- `조인 포인트(Join point)`
  - 추상적인 개념
  - `어드바이스`가 적용될 수 있는 위치, AOP가 적용될 수 있는 모든 지점
  - 스프링 AOP는 프록시 방식을 사용하기 때문에, `조인 포인트`는 메서드 실행 지점으로 제한
- `포인트컷(Pointcut)`
  - `조인 포인트` 중에서 `어드바이스`가 적용될 위치를 선별하는 기능
  - 프록시를 사용하는 스프링 AOP는 메서드 실행 시점만 `포인트컷`으로 선별 가능
- `타켓(Target)`
  - `어드바이스`를 받는 실제 객체
  - `포인트컷`으로 결정
- `어드바이스(Advice)`
  - 부가 기능
  - Around, Before, After 등 다양한 종류의 `어드바이스`가 있음
- `애스펙트(Aspect)`
  - `어드바이스` + `포인트컷`을 모듈화 한 것
  - 여러 `어드바이스`와 `포인트컷`이 함께 존재
- `어드바이저(Advisor)`
  - 하나의 `어드바이스`와 하나의 `포인트컷`으로 구성
  - 스프링 AOP에서만 사용되는 용어
- `위빙(Weaving)`
  - `포인트컷`으로 결정한 타켓의 `조인 포인트`에 `어드바이스`를 적용하는 것

<br>

## 스프링 AOP 구현

어드바이스 순서를 정할 때는 `@Order`를 사용한다.  
`@Order`는 클래스 단위로 설정할 수 있다.  
하나의 `Aspect`에 여러 개의 `어드바이스`가 있으면 순서를 정할 수 없다.

<br>

## 스프링 AOP - 포인트컷

`execution`문법

```
execution(접근제어자? 반환타입 선언타입?메서드이름(파라미터) 예외?)
```

?는 생략 가능.  
`*`와 같은 패턴을 지정할 수 있음.

<br>

## 스프링 AOP - 실무 주의사항

### 프록시와 내부 호출

스프링은 프록시 방식의 AOP를 사용한다.  
AOP를 적용하면 스프링은 대상 객체 대신 프록시 객체를 스프링 빈으로 등록한다.  
따라서, 대상 객체를 직접 호출하는 문제는 일반적으로 발생하지 않는다.  
하지만, **대상 객체의 내부에서 메서드를 호출하면 프록시를 거치지 않기 때문에 문제가 발생**한다.

<img width="596" alt="image" src="https://github.com/0takkk/inflearn/assets/89503136/5e637073-af4f-4061-b4d0-bb4da0beb4a4">
