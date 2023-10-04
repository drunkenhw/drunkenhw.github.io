---
title: '복제 지연 해결하기 (내가 원하는 db 선택하기)'
excerpt: "AOP와 Request Scope를 사용하여 원하는 데이터베이스에 접근하기"

categories:
  - java
tags:
  - java
last_modified_at: 2023-10-03T19:14:00-05:00
---

# 이 글을 쓰는 이유

저희 팀은 이전 글에서 설명했듯 database replication을 통해 부하를 분산하여 성능을 높히는 방식을 사용하고 있었습니다.
하지만 어느 날 충전소의 상태가 변경되는 것이 지연되는 걸 확인했습니다. 정확히는 source 서버에 update는 잘 작동하고 있었지만,
replica 서버에는 반영이 느린 것을 확인했습니다. 즉 **복제 지연**이 발생했던 것이 였습니다.

## 원인

아래와 같이 복제 지연이 **50519초** 발생하고 있었습니다. 게다가 저 시간이 점점 줄어야하지만 점점 늘어나는 상황이였습니다.  
![replication-lag](https://github.com/drunkenhw/comments/assets/106640954/420ebd96-96e4-40a8-b3b3-d8f1ea53b0f2){: width="500" height="100"}


복제 지연이 발생한 이유는 간단합니다. 레플리카 서버에서는 소스 서버의 데이터와 동기화를 위해 소스 서버의 binary log 파일을 읽어 동기화를 합니다. 
binary log 파일의 저장 방식은 3가지가 있습니다. **row, statement, mixed** 각각 간단히 설명드리자면
1. **row** : 행 기반의 로깅 방식으로 변경된 행들을 가지고 replica 서버에 덮어 씌우는 방식입니다.
2. **statement**: 소스 서버에 발생한 sql 명령문을 로깅하여 replica 서버에 똑같이 실행하는 방식입니다.
3. **mixed**: statement에서 발생한 `uuid()`, `now()와 같은 실행 시 마다 달라지는 쿼리가 있는 실행문은 row로 나머지는 statement로 혼합하여 저장하는 방식입니다.

이 중 저희는 row 방식을 사용하고 있었고, batch update가 잦은 저희 상황에 맞지 않았던 것입니다. 한번에 몇 천개의 데이터들이 batch update를 통해 변경되지만, 그 것들을 모두 row로 저장하고, row를 레플리카에서 변경하려다보니 지연되었던 것입니다.

그래서 저희는 statement, row 방식의 장점을 채택한 **Mixed** 방식으로 변경했고 그 후로는 지연이 일어나지 않았습니다.

## 또 다른 문제 
하지만 이렇게 수정하고도 *'복제 지연이 또 일어날 일은 없을까?'* 라는 의심이 생겼습니다.
그래서 실시간으로 정보를 보여주는 것이 가장 중요한 쿼리는 source 서버에서 읽는다면 이런 문제가 없을 것 같다는 생각이 들었습니다.
그럼 아주 간단히 변경하면 됩니다. 현재 저희 소스, 레플리카 서버를 선택하는 기준은 service 클래스에 어노테이션 `@Transactional(readOnly=true)` **read only 속성에 따라 라우팅**되고 있었습니다.

그래서 아래와 같이 어노테이션을 변경해주면 됩니다.

```java
public class StationQueryService {

    @Transactional
    public List<StationSimpleResponse> findByLocation(CoordinateRequest request, List<String> companyNames, List<ChargerType> chargerTypes, List<BigDecimal> capacities) {
        
      ...
      
        return stationQueryRepository.findStationByStationIds(stationIds);
    }
}
```
read only를 붙히지 않았습니다. 이렇게 한다면 정상적으로 해당 메서드에 사용하는 쿼리들은 source 서버에서 읽을 것입니다. 하지만 source 서버에 라우팅을 하기 위해 read only를 뗀다는 것이 이상하다고 느껴졌습니다. 왜냐면 read only는 그에 따른 역할이 충분히 있는 속성이기 때문입니다.

제가 생각한 **read only** 를 붙힘으로써 얻는 이점은 아래와 같습니다.
1. 명시적으로 개발자에게 이 메서드는 읽기만 하는 메서드라는 것을 알려줄 수 있습니다.
2. jpa 를 사용하는 경우 변경 감지 작업을 하지 않아 성능상 이점이 있습니다.
3. mysql을 사용하는 경우 read only 속성이 되어 있으면 transaction id를 부여하지 않아 성능상 이점이 있습니다. 다른 DB도 비슷한 이점이 있습니다.

이러한 이유로 **read only는 충분한 그 역할**이 있다고 생각하여 사용하지만 이번에 추가로 적용한 replication을 위해 read only 속성을 제거하는 것은 아주 이상한 일입니다.

그럼 read only는 유지하면서 제가 원하는 데이터베이스를 선택하게 할 수 없을까요?

## read only는 유지하면서 원하는 데이터베이스를 선택하기

먼저 저희 서비스의 database 라우팅 방식을 간단히 말씀드리겠습니다.
```java
public class RoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
        if (readOnly) {
            log.info("Routing to REPLICA");
            return DataSourceType.REPLICA;
        }
        log.info("Routing to SOURCE");
        return DataSourceType.SOURCE;
    }
}
```

위와 같이 `AbstractRoutingDataSource`를 상속받아 read only 속성에 따라 database에 라우팅을 하도록 만들었습니다.

그럼 간단히 해결할 수 있을 것 같습니다. 이 메서드에 무언갈 추가하여 라우팅 하는 방법을 추가하면 될 것 같습니다.

### AOP 이용하기

먼저 해당 메서드의 routing을 결정할 방법으로는 모든 메서드가 호출될 때마다 어느 database를 바라봐야하는 지 확인하면 됩니다.
하지만 이러한 방법은 아주 비효율적이고 모든 메서드에 해당 코드가 들어가게 될 것입니다. 그래서 스프링은 AOP라는 좋은 기술을 제공합니다.

>AOP(Aspect-Oriented Programming)는 핵심 로직과 부가 기능을 분리하여 애플리케이션 전체에 걸쳐 사용되는 부가 기능을 모듈화하여 재사용할 수 있도록 지원하는 것

그렇다고 합니다. 그럼 AOP를 이용해 아래와 같이 routing에 필요한 메서드를 가져와보도록 하겠습니다.

```java

@Component
@Aspect
public class RepositoryDataSourceAspect {
    
    @Pointcut("execution(public * com.carffeine.carffeine..*QueryService..*.*(..))")
    private void queryService() {
    }

    @Around("queryService()")
    public Object handler(ProceedingJoinPoint joinPoint) throws Throwable {
        Object returnType = joinPoint.proceed();
        return returnType;
    }
}

```
이렇게 된다면 클래스 이름이 QueryService라고 끝나는 모든 클래스를 PointCut으로 등록합니다. 여기서 가저온 메서드들을 어떻게 사용할 수 있을까요?

AOP에는 `@annotation()`이라는 속성으로 해당 annotation이 붙은 메서드를 가져올 수 있을 뿐만 아니라 해당 annotation의 속성까지 사용할 수 있습니다.
그럼 해당 기능을 이용해보도록 하겠습니다.

```java
public class StationQueryService {

    @Database(SOURCE)
    @Transactional(readOnly = true)
    public List<StationSimpleResponse> findByLocation(CoordinateRequest request, List<String> companyNames, List<ChargerType> chargerTypes, List<BigDecimal> capacities) {
        
      ...
      
        return stationQueryRepository.findStationByStationIds(stationIds);
    }
}
```
위와 같이 아까의 메서드에 read only 속성을 다시 부여하고 제가 만든 어노테이션인 `@Database`를 붙혔습니다.
제가 원하는 바는 해당 어노테이션의 속성 값에 따라 Source, Replica 만약 또 다른 database가 추가된다면 원하는 database의 이름을 넣어주면 해당 db로 라우팅이 되는 것입니다.
그럼 아래의 aop 메서드를 수정해보겠습니다.

```java

@Component
@Aspect
public class RepositoryDataSourceAspect {
    
    @Pointcut("execution(public * com.carffeine.carffeine..*QueryService..*.*(..))")
    private void queryService() {
    }

    @Around("queryService() && @annotation(database)")
    public Object handler(ProceedingJoinPoint joinPoint, Database database) throws Throwable {
        DatabaseType databaseType  = database.value();
        Object returnType = joinPoint.proceed();
        return returnType;
    }
}

```

이렇게 database의 값을 가져올 수 있습니다. 이제는 이 database type을 routing을 하는 조건으로 넣으면 완성입니다. 
이 메서드가 반영되어야할 부분은 하나의 스레드 즉 한번의 요청 동안 이 database type의 정보가 유지되어야 다른 클래스에서도 참조할 수 있습니다.

이러한 기능을 구현하는 방법은 Java의 **ThreadLocal**, 그리고 Spring의 **RequestScope**를 이용할 수 있을 것 같습니다. 
그 중 저는 다음과 같은 이유로 Request Scope를 사용하기로 했습니다.

1. Thread Local은 사용하고 저장되어 있는 데이터를 지워주지 않으면 다른 Thread에서 공유될 수 있습니다. 그래서 개발자가 직접 지워줘야 합니다
2. Request Scope는 Http 요청 당 하나의 Scope가 만들어지고, 요청이 끝나면 제거됩니다.
3. Thread Local을 사용하기 위해 static한 클래스로 만들게 된다면 실수로 다른 클래스에서 사용한다던지 side-effect가 생길 수 있습니다.

그럼 아래에 request scope를 이용해 저장하는 로직을 추가해보겠습니다.
```java
@Component
@Aspect
public class RepositoryDataSourceAspect {
    private final DataSoruceHolder dataSoruceHolder;
    
  ...

    @Around("queryService() && @annotation(database)")
    public Object handler(ProceedingJoinPoint joinPoint, Database database) throws Throwable {
        dataSoruceHolder.setDataSourceType(database.value());
        Object returnType = joinPoint.proceed();
        return returnType;
    }
}
```

그리고 이제 routing을 하는 클래스에 해당 request scope를 사용하도록 변경하면 끝입니다.
```java
public class RoutingDataSource extends AbstractRoutingDataSource {
    private final DataSoruceHolder dataSoruceHolder;
  ...
    @Override
    protected Object determineCurrentLookupKey() {
        if (dataSoruceHolder.isNotEmpty()) {
            DataSourceType dataSourceType = dataSoruceHolder.getDataSourceType();
            log.info("look up dataSoruce ={}", dataSourceType);
            return dataSourceType;
        }
        
        boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
        if (readOnly) {
            log.info("readOnly = true, request to replica");
            return DataSourceType.REPLICA;
        }
        log.info("readOnly = false, request to source");
        return DataSourceType.SOURCE;
    }
}
```

이제 저희가 무조건 실시간 정보를 제공해야하는 쿼리에서는 source 서버로 요청이 가게되어 복제 지연에 신경 쓸 필요가 없어졌습니다.

단점으로는 부하 분산에서 얻는 이점이 조금 줄어든다는 부분이 있을 것 같습니다.

### 결론

1. 데이터베이스 복제가 만능은 아니다.