# 스레드 로컬

## thread-per-request
- 자바에서는 동시성 처리를 하기 위해 스레드 방식을 사용해옴
	- 대표적으로 스프링 프레임워크도 멀티 스레드 모델을 사용하고 있고, 1개의 요청을 1개의 스레드가 처리하는 thread-per-request 방식으로 동작됨
![](https://i.imgur.com/qafdvUs.png)


- 이때, 자바 자체에서 스레드마다 정보를 저장할 수 있는 스레드 단위의 로컬 변수인 스레드 로컬(java.lang.ThreadLocal) 기술을 제공하고 있음
- 다시 말해, 스레드 로컬은 멀티 스레드 환경에서 각각의 스레드에게 별도의 저장소를 제공함으로서 각각의 스레드가 상태를 가질 수 있도록 도와주는 기능

 외부에 제공하는 대표적인 public 메서드 기능
 - set(T): void
 - get(): T
![](https://i.imgur.com/o54kIpY.png)



```java
// ThreadLocal#set
public void set(T value) {  
    Thread t = Thread.currentThread();  
    ThreadLocalMap map = getMap(t);  
    if (map != null) {  
        map.set(this, value);  
    } else {  
        createMap(t, value);  
    }  
}

void createMap(Thread t, T firstValue) {  
    t.threadLocals = new ThreadLocalMap(this, firstValue);  
}
```
- 현재 스레드의 키 값으로 ThreadLocalMap을 불러옴
- 없다면 ThreadLocalMap을 생성하고 값을 저장
- 있다면 ThreadLocalMap에 값을 저장

```java
// ThreadLocal#get
public T get() {  
    Thread t = Thread.currentThread();  
    ThreadLocalMap map = getMap(t);  
    if (map != null) {  
        ThreadLocalMap.Entry e = map.getEntry(this);  
        if (e != null) {  
            @SuppressWarnings("unchecked")  
            T result = (T)e.value;  
            return result;  
        }  
    }  
    return setInitialValue();  
}

ThreadLocalMap getMap(Thread t) {  
    return t.threadLocals;  
}
```
- 현재 스레드의 키 값으로 ThreadLocalMap을 불러옴
- 없다면 ThreadLocalMap을 생성 후 빈 값 반환
- 있다면 ThreadLocalMap.Entry를 가져와서 ThreadLocal에 저장한 값을 불러옴


---


## 사용 예시
- 쿠키에서 사용자 세션 정보 조회
- 세션 정보를 스레드 로컬에 저장
- 요청 처리가 끝나면 스레드 로컬 정리
![](https://i.imgur.com/0RkeBxY.png)


```java
@Component  
public class SessionInterceptor implements HandlerInterceptor {  
  
    @Override  
    public boolean preHandle(  
            final HttpServletRequest request,  
            final HttpServletResponse response,  
            final Object handler  
    ) throws Exception {  
        final HttpSession session = CookieUtil.getSession(request);  
        SessionContextHolder.set(session);  
  
        return true;  
    }  
    ...
}
```

```java
public class SessionContextHolder {  
  
    private static final ThreadLocal<HttpSession> contextHolder;  
  
    static {  
        contextHolder = new ThreadLocal<>();  
    }  
  
    public static void set(final HttpSession session) {  
        contextHolder.set(session);  
    }  
  
    public static HttpSession get() {  
        return contextHolder.get();  
    }  
  
    public static void unset() {  
        contextHolder.remove();  
    }  
  
}
```

```java
@RestController  
public class OrderController {  
  
    private final Logger logger = Logger.getLogger(this.getClass().getName());  
  
    @PostMapping("/api/v1/orders")  
    public void createOrder() {  
		
		logger.info("[주문 요청] userId = " + httpSession.getId());  
		
        // ThreadLocal에서 사용자 정보 조회  
        final HttpSession httpSession = SessionContextHolder.get();  
        
        // something...
        
        logger.info("[주문 완료] userId = " + httpSession.getId() +   
                ", orderId = " + UUID.randomUUID());  
    }  
  
}
```

```java
@Component  
public class SessionInterceptor implements HandlerInterceptor {  
	...
	
    @Override  
    public void afterCompletion(  
            HttpServletRequest request,  
            HttpServletResponse response,  
            Object handler,  
            Exception ex  
    ) throws Exception {  
    
        // ThreadLocal 정리  
        SessionContextHolder.unset();  
    }  
}
```


### 스프링 프레임워크에서의 ThreadLocal
### Spring Security SecurityContextHolder

![](https://i.imgur.com/PKSbfFH.png)
- 스프링 시큐리티에서는 SecurityContextHolder안에 Authentication 정보를 담게 되는데 각 요청 스레드마다 SecurityContext를 저장하거나 가져오는데 스레드 로컬이 사용됨


### Spring Transactional TransactionSynchronizationManager
![](https://i.imgur.com/J6TIxY1.png)
- Transactional Annotation은 트랜잭션에 대한 정보들을 ThreadLocal로 관리


## 스레드 풀과 함께 사용할 때 주의점
일반적으로 스프링 프레임워크를 사용하면 기본 스레드 풀인 내장 톰캣의 스레드 풀을 사용하게 되는데, 이러한 스레드 풀을 사용하는 환경에서는 스레드 로컬의 사용을 주의해야 한다.

**스레드 로컬**
- 각 스레드에게별도의 저장소를 제공하는 기능
- 이 말은 즉, 스레드가 재활용될 수 있고 그에 따라 스레드 로컬에 누군가(?)의 정보가 남아 있을 수 있음을 의미함

```java
@Test  
@DisplayName("스레드 풀 환경에서 ThreadLocal을 지웠을 때 문제가 발생하지 않는 예시")  
void removeNotWithThreadPool() {  
  
    final int threadCount = 5;     // 실행할 작업의 총 개수  
    final int threadPoolSize = 3;  // 스레드 풀의 스레드 개수  
  
    ExecutorService executorService = Executors.newFixedThreadPool(threadPoolSize);  // 3개의 스레드를 가진 스레드 풀 생성  
  
    for (int i = 0; i < threadCount; i++) {  
  
        // given: 세션 생성  
        HttpSession session = mock(HttpSession.class);  
        when(session.getId()).thenReturn(UUID.randomUUID().toString());  
  
        executorService.submit(() -> {  
  
            final String threadName = Thread.currentThread().getName().substring(7);  
  
            // when & then: ThreadLocal 정보 출력  
            System.out.printf("[%s]  started: SESSION ID = %s%n", threadName, SessionContextHolder.get());  
            SessionContextHolder.set(session);  
            System.out.printf("[%s] finished: SESSION ID = %s%n", threadName, SessionContextHolder.get());  
            SessionContextHolder.unset();  --> 추가
        });  
    }  
    // Executor 서비스 종료  
    executorService.shutdown();  
}
```

```
[thread-1]  started: SESSION ID = null
[thread-3]  started: SESSION ID = null
[thread-2]  started: SESSION ID = null
[thread-3] finished: SESSION ID = Mock for HttpSession, hashCode: 1095849663
[thread-2] finished: SESSION ID = Mock for HttpSession, hashCode: 832066800
[thread-1] finished: SESSION ID = Mock for HttpSession, hashCode: 866901553
[thread-3]  started: SESSION ID = null
[thread-2]  started: SESSION ID = null
[thread-3] finished: SESSION ID = Mock for HttpSession, hashCode: 1257532915
[thread-2] finished: SESSION ID = Mock for HttpSession, hashCode: 1096343229
```


---



```java
@Test  
@DisplayName("스레드 풀 환경에서 ThreadLocal을 지우지 않아 문제가 발생하는 예시")  
void removeWithThreadPool() {  
  
    final int threadCount = 5;     // 실행할 작업의 총 개수  
    final int threadPoolSize = 3;  // 스레드 풀의 스레드 개수  
  
    ExecutorService executorService = Executors.newFixedThreadPool(threadPoolSize);  // 3개의 스레드를 가진 스레드 풀 생성  
  
    for (int i = 0; i < threadCount; i++) {  
  
        // given: 세션 생성  
        HttpSession session = mock(HttpSession.class);  
        when(session.getId()).thenReturn(UUID.randomUUID().toString());  
  
        executorService.submit(() -> {  
  
            final String threadName = Thread.currentThread().getName().substring(7);  
  
            // when & then: ThreadLocal 정보 출력  
            System.out.printf("[%s]  started: SESSION ID = %s%n", threadName, SessionContextHolder.get());  
            SessionContextHolder.set(session);  
            System.out.printf("[%s] finished: SESSION ID = %s%n", threadName, SessionContextHolder.get());  
        });  
    }  
    // Executor 서비스 종료  
    executorService.shutdown();  
}
```

```
[thread-1]  started: SESSION ID = null
[thread-2]  started: SESSION ID = null
[thread-3]  started: SESSION ID = null
[thread-1] finished: SESSION ID = Mock for HttpSession, hashCode: 866901553
[thread-2] finished: SESSION ID = Mock for HttpSession, hashCode: 832066800
[thread-3] finished: SESSION ID = Mock for HttpSession, hashCode: 1095849663
[thread-1]  started: SESSION ID = Mock for HttpSession, hashCode: 866901553
[thread-2]  started: SESSION ID = Mock for HttpSession, hashCode: 832066800
[thread-1] finished: SESSION ID = Mock for HttpSession, hashCode: 1257532915
[thread-2] finished: SESSION ID = Mock for HttpSession, hashCode: 1096343229
```


참고
- https://velog.io/@skygl/ThreadLocal