---
title: '캐시와 이분 탐색으로 조회 성능 개선하기'
excerpt: "조회 성능 개선하기 3"

categories:
  - java
tags:
  - java
last_modified_at: 2023-09-17T19:14:00-05:00
---

# 이 글을 쓰는 이유
이전 글에서도 계속 설명했듯이 조회 성능을 최대한 빠르게 하는 것이 저희 서비스에서 핵심이라고 생각하기 때문에 지금도 예전에 비해 빨라졌지만 다른 개선점이 보여 개선을 하고자합니다.

[조회 성능 개선하기 1 (인덱스)](https://drunkenhw.github.io/mysql/improved-query-performance/)

[조회 성능 개선하기 2 (데이터베이스 복제)](https://drunkenhw.github.io/mysql/db-replication/)
## 결론
결론부터 말씀드리면 로컬에서 캐싱을 적용한 후 100명의 사용자가 지도의 데이터를 조회할 때를 기준으로

**TPS** 78 -> 128

**Response Time** 1236 ms -> 751 ms

약 **64%** 성능이 개선 되었습니다.

*(저번 성능 테스트의 결과가 다른 이유는 비즈니스 로직이 변경되어 조회 방식이 바뀌었기 때문입니다. 그래서 캐싱을 적용하기전, 한 후 를 비교했습니다.)*

# Caching

>In computing, a cache is a hardware or software component that stores data so that future requests for that data can be served faster; the data stored in a cache might be the result of an earlier computation or a copy of data stored elsewhere.

캐싱은 위키 백과에서 위와 같이 설명하고 있습니다. 즉 메모리에 데이터를 복사본을 올려 좀 더 빠르게 데이터에 접근하는 방식입니다.

캐싱의 단점은 수정, 삽입, 삭제가 되었을 때, 관리 포인트가 두 군데가 된다는 점입니다. 만약 데이터베이스에만 새로운 정보를 저장하고, 캐시에는 저장해주지 않는다면 사용자는 그 정보를 볼 수 없습니다.

하지만 저희 서비스에서 적용한 이유는 충전기의 충전 상태 (충전 중, 대기중, 고장)에 대한 정보는 최신화가 되어야하지만, 충전소의 이름이라던지, 위치, 다른 정보들은 쉽게 변하지 않기 때문에 해당 정보를 캐싱한다면 좋을 것 같았습니다.

# 캐싱 적용하기

먼저 캐싱을 어디에서 하는지도 중요합니다. 크게 **로컬 캐시**와 **글로벌 캐시**로 나눌 수 있을 것 같습니다.
글로벌 캐시의 장점은 스케일 아웃을 했을 때, 모든 서버가 다 같은 데이터를 바라보기 때문에 데이터 정합성이 좋아집니다. 하지만 저희 서비스는 단일 서버로 구성되어 있기 때문에, 로컬 캐시를 해도 문제가 없습니다. 그리고 글로벌 캐시를 적용하기 위해서는 Redis나 Memcached 같은 도구를 모든 팀원이 알아야하지만 로컬 캐시는 그렇게 하지 않더라도 편하게 적용할 수 있다는 점에서 로컬에 캐싱하는 방법을 적용해보겠습니다.

## 캐싱할 정보 가져오기

캐싱을 하기 위해서는 먼저 캐싱할 데이터를 가져와야합니다. 저희 서비스는 출장 혹은 여행을 가는 전기차 오너가 핵심 페르소나이기 때문에 사용자들이 찾는 정보의 위치는 불특정합니다. 서울에서 다른 지방으로 출장을 가는 경우도 있을 것이고, 지방에서 서울에 가는 경우도 있기 때문에, 모든 데이터를 캐싱해야할 것이라 판단했습니다.

그래서 어플리케이션 실행 시에 모든 충전소를 캐싱하기로 선택했습니다.

```java
@Configuration
public class InitialStationCache implements ApplicationRunner {

    private final StationCacheRepository stationCacheRepository;
    private final StationQueryRepository stationQueryRepository;

    @Override
    public void run(ApplicationArguments args) {
        log.info("Initialize station cache");
        List<StationInfo> stations = stationQueryRepository.findAll();
        stationCacheRepository.initialize(stations);
        log.info("Station cache initialized");
        log.info("Station cache size: {}", stations.size());
    }
}

```

ApplicationRunner를 구현하여 어플리케이션 실행 시 모든 충전소의 정보를 가져오도록 만들었습니다.
여기서 Entity인 Station을 가져오지 않은 이유는 크게 두가지가 있습니다. 
1. 지도로 조회하는 부분의 성능을 개선하고자 했지만, Entity에는 지도를 조회할 때 불필요한 정보도 있기 때문에 메모리상의 낭비가 생길 수 있습니다.
2. Entity를 캐싱하게 된다면 hibernate 1차 캐시에도 적재되고, 힙 메모리에도 적재되는 일이 발생하여 메모리상 낭비라고 생각했습니다.

## 범위 검색하기

충전소의 데이터를 조회하는 조건은 위도, 경도의 최소, 최대값을 기준으로 만족하는 데이터를 보여줍니다.
아래와 같이 간단히 조건을 stream()의 filter()를 사용해서 구현했습니다.
```java
public class StationCacheRepository {
    
    private final List<StationInfo> cachedStations;
    
    public List<StationInfo> findByCoordinate(
          BigDecimal minLatitude,
          BigDecimal maxLatitude,
          BigDecimal minLongitude,
          BigDecimal maxLongitude
  ) {
   return cachedStations.stream()
                .filter(it -> it.latitude().compareTo(minLatitude) >= 0 && it.latitude().compareTo(maxLatitude) <= 0)
                .filter(it -> it.longitude().compareTo(minLongitude) >= 0 && it.longitude().compareTo(maxLongitude) <= 0)
                .toList();
  }
}
```
하지만 해당 방법으로 로컬에서 조회를 테스트 했을 때 캐시를 적용한 것보다 더 느려진 결과가 나왔습니다.
캐싱을 해서 데이터베이스까지 요청을 보내지 않는데 왜 더 느려진 것일까요?

답은 **인덱스** 였습니다. Mysql 에서 인덱스는 B Tree로 구성되어 있습니다. 데이터베이스에서는 위도, 경도로 복합 인덱스가 설정되어 있었지만, 현재 어플리케이션 로직에는 해당 부분이 없습니다.

그래서 filter로 순회하는 시간복잡도가 O(n)이고, 데이터베이스에서는 O(log n)이기 때문에 더 느려진 것입니다. 그렇다고 제가 직접 B tree 자료구조를 직접 구현해야할까요?

현재 해당 조회 API는 위도 경도로 범위 탐색을 하고 있습니다. 결국엔 station의 정보들이 위도, 경도로 정렬만 되어 있다면 B tree를 직접 구현하지 않더라도 같은 시간복잡도 O(log n)으로 탐색할 수 있습니다.
물론 B tree와 다른 부분은 해당 충전소의 정확한 위도, 경도로 단일 칼럼을 조회할 때는 O(n)이기 때문에 이런 방법이 문제가 될 수 있지만, 해당 캐시 데이터로는 무조건 범위 탐색을 하기 때문에, B tree를 구현하지 않고 이분 탐색으로 조회하는 방식으로 변경해보겠습니다.

```java
    public void initialize(List<StationInfo> stations) {
        cachedStations.addAll(stations);
        cachedStations.sort((o1, o2) -> {
             int latitudeCompare = o1.latitude().compareTo(o2.latitude());
             if (latitudeCompare == 0) {
                  return o1.longitude().compareTo(o2.longitude());
             }
             return latitudeCompare;
        });
    }
    
    private List<StationInfo> findStations(BigDecimal minLatitude, BigDecimal maxLatitude, BigDecimal minLongitude, BigDecimal maxLongitude) {
        int lowerBound = binarySearch(minLatitude, START_INDEX);
        int upperBound = binarySearch(maxLatitude, lowerBound);
        if (lowerBound == -1 || upperBound == -1) {
          return Collections.emptyList();
        }
        return cachedStations.stream()
                      .skip(lowerBound)
                      .limit(upperBound - lowerBound)
                      .filter(station -> station.longitude().compareTo(minLongitude) >= 0 && station.longitude().compareTo(maxLongitude) <= 0)
                      .toList();
    }
    
    private int binarySearch(BigDecimal latitude, int startIndex) {
        int left = startIndex;
        int right = cachedStations.size() - 1;
        int result = -1;
        while (left <= right) {
            int middle = left + (right - left) / 2;
            StationInfo middleStation = cachedStations.get(middle);
            if (middleStation.latitude().compareTo(latitude) >= 0) {
                result = middle;
                right = middle - 1;
            } else {
                left = middle + 1;
            }
        }
        return result;
    }

```

먼저 어플리케이션이 실행될 때 cache 데이터를 찾아 저장하는 것 뿐만 아니라, 위도(Latitude)를 기준으로 정렬하도록 만들었습니다. 그리고 위도의 최소, 최대값의 인덱스를 가장 효율적으로 찾아올 수 있도록 binary search를 하는 메서드를 만들었습니다. 이렇게 한다면 O(log n) 으로 위도의 최대 최소 조건에 포함되는 모든 station의 값을 조회할 수 있습니다. 그리고 조회한 데이터들의 개수만큼 filter를 통해 경도(longitude) 가 포함되는지 확인합니다. 해당 방식의 구현은 B tree가 작동하는 방식과 유사할 것입니다. 

이분 탐색을 적용한 결과 로컬에서 응답 속도가 120 ms -> 50 ~ 70 ms로 약 2배 빨라진 것을 확인할 수 있습니다.

## 실시간이 중요한 데이터는?

앞서 말씀드렸다시피 지도로 충전소를 조회할 때, 충전소의 정보들에는 바뀌지 않는 정보뿐만 아니라, 최신화해야하는 충전기의 현재 상태 정보가 있습니다. 이러한 정보들은 캐싱해둘 수 없습니다. 하더라도, 관리 포인트가 늘어나기 때문에 데이터베이스에서 캐싱해둔 충전기 id로 충전기의 상태를 찾아와서 정보를 합쳐 반환하는 식으로 만들 수 있습니다.
```sql
    select cs.station_id,
           sum(case
                   when cs.charger_condition = 'STANDBY' then 1
                   else 0
               end)               
    from charger_status cs
    where cs.station_id in (?, ?, ?, ?, ?, ?, ?)
    group by cs.station_id
```
위와 같은 쿼리로 해당 충전소의 최신화된 충전기 상태를 가져올 수 있습니다.

캐싱을 하기전에 데이터베이스를 이용해 데이터를 가져올 때의 쿼리는 아래와 같습니다.

```sql
 select
        distinct s.station_id  
    from
        charge_station s 
    inner join
        charger c 
            on (
                c.station_id=s.station_id
            ) 
    where
        s.latitude>=? 
        and s.latitude<=? 
        and s.longitude>=? 
        and s.longitude<=?
 -------------------------------------------------
    select
        s.station_id,
        s.station_name,
        s.latitude,
        s.longitude,
        s.is_parking_free,
        s.is_private,
        sum(case 
            when cs.charger_condition='STANDBY' then 1 
            else 0 
        end),
        sum(case 
            when c.capacity>=50 then 1 
            else 0 
        end) 
    from
        charge_station s 
    inner join
        charger c 
            on (
                c.station_id=s.station_id
            ) 
    inner join
        charger_status cs 
            on (
                c.station_id=cs.station_id 
                and c.charger_id=cs.charger_id
            ) 
    where
        s.station_id in (
            ?,?,?,?
        ) 
    group by
        s.station_id
```
원래는 위와 같이 여러번의 Join을 하고, 2번의 쿼리가 나갔던 반면 지금은 join을 하지않는 한번의 깔끔한 쿼리로 개선되었습니다.

그리고 **station 테이블의 위도, 경도로 범위 탐색을 위해 생성했던 index도 제거**할 수 있게 되었습니다!

## 결론
1. 캐싱할 수 있는 부분은 하는 것도 좋을 것 같습니다
2. 시간 복잡도를 계산해봅시다.
3. 성능 개선 재밌습니다.
