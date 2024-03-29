---
title: '쿼리 성능 개선하기'
excerpt: "실행 계획을 확인하자"

categories:
  - mysql
tags:
  - mysql
last_modified_at: 2023-09-03T19:14:00-05:00
---

# 이 글을 쓰는 이유

먼저 이 글을 쓰게 된 계기를 말씀드리겠습니다. 카페인 팀 프로젝트에는 사용자가 보고있는 지도에 충전소를 보여주는 조회 기능이 가장 중요하고, 제일 요청이 많이 들어옵니다.

하지만 조회 성능이 좋지 않은 까닭인지 여러 사용자가 접속하면 아래와 같이 데이터베이스가 실행되고 있는 서버의 cpu 사용률이 100%가 되는 문제가 있었습니다.
![cpu](https://github.com/drunkenhw/drunkenhw.github.io/assets/106640954/2330435f-17b4-4d38-b16b-c72fd7017969)

# 조회 성능 개선하기

먼저 제가 개선하기 위해 사용했던 방법들에 대해 적어보겠습니다.

## 1. DTO 이용하기

현재 구조는 아래의 JPA를 이용해 아래와 같은 쿼리로 entity로 데이터를 조회합니다.

```sql
 select distinct station.station_id,
                 charger.charger_id,
                 charger.station_id,
                 chargerStatus.charger_id,
                 chargerStatus.station_id,
                 station.created_at,
                 station.updated_at,
                 station.address,
                 station.company_name,
                 station.contact,
                 station.detail_location,
                 station.is_parking_free,
                 station.is_private,
                 station.latitude,
                 station.longitude,
                 station.operating_time,
                 station.private_reason,
                 station.station_name,
                 station.station_state,
                 charger.created_at,
                 charger.updated_at,
                 charger.capacity,
                 charger.method,
                 charger.price,
                 charger.type,
                 charger.station_id,
                 charger.charger_id,
                 chargerStatus.created_at,
                 chargerStatus.updated_at,
                 chargerStatus.charger_condition,
                 chargerStatus.latest_update_time
 from charge_station station
        inner join
      charger charger on station.station_id = charger.station_id
        inner join
      charger_status chargerStatus on charger.charger_id = chargerStatus.charger_id
        and charger.station_id = chargerStatus.station_id
 where station.latitude >= 37.5019194727953082567
   and station.latitude <= 37.5092305272047217433
   and station.longitude >= 127.044542269049714936
   and station.longitude <= 127.058071330950285064

```

JPA를 통해 이러한 방식으로 조회한다면 아주 편하게 값을 가져오고, fetch join을 통해 하위의 entity들의 정보도 깔끔하게 가져옵니다.

가져온 값으로 필요한 정보들을 매핑하고 가공하여 응답을 내려줬습니다.

하지만 조회만을 위해 JPA의 entity를 조회한다는 것은 여러 단점이 존재합니다.

제일 먼저 응답을 내려줄 때 불필요한 데이터까지 모두 조회를 한다는 부분입니다.
이렇게 많은 필드들이 있습니다. 하지만 응답에서는 대부분의 경우 모든 정보가 필요하지 않습니다. 그리고 모든 정보를 다 보내주는 것도 좋지 않습니다. 대량의 데이터를 조회할 때의 성능이 아주 나빠집니다.

그래서 필요한 칼럼만 조회하는 것이 좋습니다.

그리고 또 다른 단점으로는 JPA로 entity를 조회할 때 Hibernate 캐시에 저장한다던가, One To One 에서 N+1 쿼리가 발생하기 때문에 성능적인 이슈가 여러가지 있습니다.

그래서 조회만 하는 api라면 DTO Projection으로 하는 것이 좋을 것 같습니다.
그리고 아래와 같이 변경하였습니다.

```sql
SELECT s.station_id,
       s.station_name,
       s.latitude,
       s.longitude,
       s.is_parking_free,
       s.is_private,
       sum(case
               when cs.charger_condition = 'STANDBY' then 1
               else 0
           end),
       sum(case
               when c.capacity >= 50 then 1
               else 0
           end)
FROM charge_station s
         inner join charger c on (c.station_id = s.station_id)
         inner join charger_status cs on (c.charger_id = cs.charger_id and c.station_id = cs.station_id)
where s.station_id in (?, ?)
group by s.station_id;
```

이렇게 필요한 칼럼만 조회하는 방식으로 변경하여, 선릉역 근처를 조회하는 기준으로 약 450ms -> 350ms로 개선되었습니다.

하지만 아직도 너무 느린 것을 확인할 수 있습니다. 그래서 실행 계획을 확인했습니다. 

## 실행 계획 확인하기

sql의 실행 계획은 아주 중요하고 성능을 개선할 때 아주 유용합니다.

실행 계획에는 여러가지 정보들이 있습니다.
1. ID: 실행 계획 내에서 각 작업 또는 단계를 식별하는 일련번호입니다. 실행 계획은 여러 단계로 나뉘며, ID를 통해 이러한 단계를 식별할 수 있습니다.

2. Select Type: 쿼리의 각 단계(예: SIMPLE, PRIMARY, SUBQUERY)에 대한 실행 유형을 나타냅니다. 이는 MySQL이 데이터를 선택하고 처리하는 방식을 나타냅니다.

3. Table: 실행 계획에 포함된 테이블의 이름 또는 별칭입니다. 어떤 테이블이 사용되는지를 확인할 수 있습니다.

4. Type: 테이블 접근 방식을 나타냅니다. 이 값은 인덱스 스캔, 풀 테이블 스캔 등과 같은 값일 수 있으며, 성능에 큰 영향을 미칩니다.

5. Possible Keys: 사용 가능한 인덱스를 나타냅니다. MySQL이 어떤 인덱스를 사용할 수 있는지 알려줍니다.

6. Key: 실제로 선택된 인덱스입니다. 이 값은 가능한 인덱스 중에서 실제로 사용되는 인덱스를 나타냅니다.

7. Key Len: 사용된 인덱스의 길이를 나타냅니다.

8. Ref: 인덱스를 사용하여 테이블 간의 연결을 나타내는 열입니다.

9. Rows: 각 단계에서 예상되는 행의 수입니다. 이 값은 성능 평가에 중요한 역할을 합니다.

10. Extra: 기타 정보를 제공합니다. 이 칼럼에는 추가 정보 및 힌트가 포함될 수 있습니다.

이렇게 여러 칼럼이 있습니다. 그 중 성능에 큰 영향을 미치는 칼럼 두 가지만 자세히 알아보겠습니다.

### Type
1. **const** : 쿼리에 Primary key 혹은 unique key 칼럼을 이용하는 where 조건절을 가지고 있고, 반드시 하나의 데이터를 반환하는 방식이다. (옵티마이저가 해당 부분은 상수로 처리하기 때문에 const라고 한다.)
2. **eq_ref** : 조인에서 Primary key 혹은 unique key 칼럼을 이용하는 where 조건절을 가지고 있고, 반드시 하나의 데이터를 반환하는 방식이다. (const와 다른 점은 eq_ref는 조인에서 사용된다는 점이다.)
3. **ref** : eq_ref와 다르게 join의 순서와 관계없이 사용된다. 그리고 primary key, unique key도 관계없다. 그냥 인덱스의 종류와 관계없이 `=` 조건으로 검색할 때 사용된다
4. **fulltext**:  mysql 전문 검색 인덱스를 사용해서 레코드에 접근하는 방법, 전문 검색할 컬럼에 인덱스가 있어야 한다. "MATCH ... AGAINST ..." 구문을 사용해서 실행된다
5. **range**: 인덱스를 이용해서 검색하는데, 검색 조건이 `>, >=, <, <=, BETWEEN, IN()` 등의 연산자를 사용하는 경우이다. 보통의 인덱스 스캔이라고 하면 range, const, ref를 칭한다
6. **index**: 인덱스 풀 스캔이다. 인덱스를 이용해서 테이블의 모든 레코드를 읽는다. 인덱스를 이용해서 테이블을 읽는 것이기 때문에 all보다는 빠르다. 
7. **all**: 테이블 풀 스캔이다. 테이블의 모든 레코드를 읽는다. 가장 느린 방법이다.

실행 계획에서 자주 보이는 type들만 **성능이 좋은 순**으로 정리해봤습니다.

### Extra
1. **using filesort**: 정렬을 위해 별도의 파일 정렬을 수행한다. 이는 인덱스를 사용하지 않고 정렬을 수행한다는 의미이다. 이는 성능에 좋지 않다.
2. **using index**: 인덱스만으로 쿼리를 처리한다. 이는 인덱스만으로 쿼리를 처리하기 때문에 성능이 좋다.
3. **using join** buffer: join이 되는 칼럼은 인덱스를 생성한다. 하지만 driven table에 적절한 인덱스가 없다면 driving table에 있는 모든 레코드를 읽어서 join을 수행한다. 그래서 이걸 보완하기 위해 driving table에 읽은 레코드를 임시 공간에 저장하는데 그 곳이 join buffer이다.
4. **using temporary**: 쿼리를 처리하기 위해 임시 테이블을 생성한다. 인덱스를 사용하지 못하는 group by 쿼리가 대표적인 예이다.
5. **using where**:  mysql 엔진이 별도의 가공, 필터링 작업을 처리한 경우일 때만 나타난다. 범위 조건은 스토리지 엔진에서 처리되어 레코드를 리턴해주지만, 체크 조건은 mysql 엔진에서 처리된다.


type뿐만 아니라 extra도 쿼리의 문제를 파악하는데 아주 큰 도움을 줍니다. 그 중 자주 보이는 것들에 대해서만 정리해봤습니다.

그럼 아까 생성한 쿼리의 실행 계획을 확인해봅시다.
```
+----+-------------+--------------+------------+--------+----------------------------------+---------+---------+---------------------------------------------------------+--------+----------+--------------------------------------------+
| id | select_type | table        | partitions | type   | possible_keys                    | key     | key_len | ref                                                     | rows   | filtered | Extra                                      |
+----+-------------+--------------+------------+--------+----------------------------------+---------+---------+---------------------------------------------------------+--------+----------+--------------------------------------------+
|  1 | SIMPLE      | station      | NULL       | range  | PRIMARY,idx_station_coordination | PRIMARY | 1022    | NULL                                                    |      2 |   100.00 | Using where; Using temporary               |
|  1 | SIMPLE      | charger      | NULL       | ALL    | PRIMARY                          | NULL    | NULL    | NULL                                                    | 240340 |    10.00 | Using where; Using join buffer (hash join) |
|  1 | SIMPLE      | chargersta   | NULL       | eq_ref | PRIMARY                          | PRIMARY | 2044    | charge.charger1_.charger_id,charge.station0_.station_id |      1 |   100.00 | NULL                                       |
+----+-------------+--------------+------------+--------+----------------------------------+---------+---------+---------------------------------------------------------+--------+----------+--------------------------------------------+
```

station 테이블에 대해서는 range 스캔, 임시 테이블을 생성하고 있습니다, 그리고 charger에서는 테이블 풀 스캔, join buffer까지 생성하고 있습니다. 다행히도 chargersta 테이블에서는 적당한 조건을 생성한 것 같습니다.

다시 한번 쿼리를 보고 실행 계획이 이렇게 나온 이유를 알아보겠습니다.
```sql
SELECT
    ...
    FROM charge_station s
    inner join charger c on (c.station_id = s.station_id)
    inner join charger_status cs on (c.charger_id = cs.charger_id and c.station_id = cs.station_id)
where s.station_id in (?, ?)
group by s.station_id;
```

아까 얘기했던, using temporary와 using join buffer가 발생하는 이유의 공통점을 찾아보면, 인덱스가 문제인 것을 유추할 수 있습니다.

station과 charger를 join할 때, driven table 즉, charger 테이블에 적절한 인덱스가 없어 성능이 나빠진 것이라 의심하여, 인덱스를 생성하고 다시 한번 실행 계획을 확인했습니다.

```
+----+-------------+--------------+------------+--------+----------------------------------+-------------------+---------+---------------------------------------------------------+--------+----------+---------------+
| id | select_type | table        | partitions | type   | possible_keys                    | key               | key_len | ref                                                     | rows   | filtered | Extra         |
+----+-------------+--------------+------------+--------+----------------------------------+-------------------+---------+---------------------------------------------------------+--------+----------+---------------+
|  1 | SIMPLE      | station      | NULL       | range  | PRIMARY,idx_station_coordination | PRIMARY           | 1022    | NULL                                                    |      2 |   100.00 | Using where   |
|  1 | SIMPLE      | charger      | NULL       | ref    | PRIMARY,idx_station_id           | idx_station_id    | 1022    | charge.s.station_id                                     |      3 |   100.00 | NULL          |
|  1 | SIMPLE      | chargersta   | NULL       | eq_ref | PRIMARY                          | PRIMARY           | 2044    | charge.charger1_.charger_id,charge.station0_.station_id |      1 |   100.00 | NULL          |
+----+-------------+--------------+------------+--------+----------------------------------+-------------------+---------+---------------------------------------------------------+--------+----------+---------------+
```

이렇게 charger 테이블에 인덱스를 생성한 것만으로도 실행 계획을 깔끔하게 개선했습니다.

### 결과
아래는 인덱스를 생성하기 전 실행 속도입니다.

![개선_전](https://github.com/woowacourse-teams/2023-car-ffeine/assets/106640954/1130eee6-c2b9-4846-b294-73de78b0f070)

아래는 인덱스를 생성한 후 실행 속도입니다.

![개선_후](https://github.com/woowacourse-teams/2023-car-ffeine/assets/106640954/d024330a-c233-4e75-a28b-1b01b6ae3245)

315ms -> 24ms 로 약 13배 빨라진 것을 확인할 수 있습니다.

# 결론

**실행 계획 확인은 필수입니다!**   

### 참고
real mysql 책 



