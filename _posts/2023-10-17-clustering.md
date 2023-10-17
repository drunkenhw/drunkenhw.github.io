---
title: '범위 내 충전소 마커 클러스터링 하기'
excerpt: '범위 내 충전소 마커 클러스터링 하기'

categories:
  - java
tags:
  - java
last_modified_at: 2023-10-17T19:14:00-05:00
---

# 이 글을 쓰는 이유

저희 카페인 팀의 서비스는 전기차 충전소를 조회가 핵심입니다. 전국에 충전소는 약 6만건, 충전기는 약 23만건 이상 있습니다. 서울시가 보이는 범위의 지도를 조회하게 된다면 약 2.5초 이상의 응답 시간이 걸립니다.
![time](https://github.com/woowacourse-teams/2023-car-ffeine/assets/106640954/63ec647e-83f7-4137-b813-c63606aca7e8)

이렇게 된다면 사용자는 지도를 축소할 때마다 불편함을 느낄 것이고, 마커들이 개미처럼 잔뜩 보여 클릭하기도 힘든 환경일 것입니다.

따라서 저희 팀에서는 임시 방편으로 지도를 어느정도 축소하게 된다면 지도를 확대해달라는 문구와 함께 요청을 보내지 않는 식으로 만들었습니다.

![asis](https://github.com/woowacourse-teams/2023-car-ffeine/assets/106640954/8792447a-5e3b-4afe-b556-2ef20e1c9cfd){: width="200" height="100"}

하지만 이러한 문제 때문에 지도 조차 보기 힘듭니다. 사용자 피드백에서 제일 많이 언급되었고, 이 부분을 해결하고자 했던 일들을 적어보려 합니다.

**결론은 오직 java만 사용해 클러스터링을 하였습니다.**

## Mysql Point Type으로 변경하기 

Mysql과 몇몇의 DB에서 지원하는 Geometry Type이 있습니다. 여기에는 여러가지 타입들이 있습니다.
1. Point : 좌표 공간의 한 점
2. LineString : 여러 개의 Point를 연결해 주는 선
3. Polygon : 다각형

그리고 아래와 같은 편한 특수 함수들을 지원합니다. 대표적인 함수는 다음과 같습니다.
1. ST_GeomFromText(WKT, SRID) : WKT와 SRID로 Geometry를 만듭니다
2. ST_Buffer(Point, Radius) : Point를 중심으로 Radius를 그립니다.
3. ST_Contains(x, y): y가 x에 포함되어 있는지 확인합니다
4. ST_Within(x, y): x가 y에 포함되어 있는지 확인합니다.
5. ST_Distance(x, y): x와 y 사이의 거리를 반환합니다

이러한 유용한 기능들을 사용할 수 있고, 저장할 수 있습니다. 그리고 공간 인덱스인 R-Tree를 이용하기 때문에 공간 검색에 대해서는 B Tree보다 성능이 좋습니다. 자세한 내용은 [링크](https://en.wikipedia.org/wiki/R-tree)에서 확인 부탁드리겠습니다

그래서 Mysql의 Point Type과 Polygon을 적용한다면, 적절한 크기의 다각형에 충전소의 위치들을 저장할 수 있겠다 싶었습니다.
그리고 공간 인덱스인 R-Tree를 적용하면 속도도 더 빨라질 것 같았습니다.

그래서 BigDecimal로 위도 경도가 저장되어있던 것을 Point 타입으로 마이그레이션 시작했습니다.

먼저 Point Type은 JPA에서 기본적으로 제공하지 않기 때문에 외부 라이브러리의 의존성을 추가해야합니다.
```groovy
implementation 'org.hibernate.orm:hibernate-spatial:{hibernate vesion}'
```

해당 라이브러리를 추가하면 아래와 같이 클래스를 추가만들 수 있습니다.
```java
import org.locationtech.jts.geom.Point;

@Embaddalbe
public class StationPoint {
    private Point point;
}
```
이제 이런 방식으로 마이그레이션만 하면 끝입니다. 
하지만 저희 팀은 JDBC를 사용하여 저장하는 경우가 많았습니다. 문자열인 JDBC sql 구문을 위해 모든 sql을 하나하나 검사해서 약 10개의 메서드를 수정했습니다.

이제 조회 메서드만 수정하면 끝입니다. 조회는 Query Dsl을 사용하고 있었기 때문에 또 다른 의존성을 추가해줘야 합니다.
```groovy
implementation 'com.querydsl:querydsl-sql-spatial:5.0.0'
```
querydsl에서 spatial 타입을 사용할 수 있도록하는 의존성입니다. 그리고 이제 조회만 하면 끝입니다. **하지만 test가 전부 실패했습니다.**

이유는 h2에서 공간 데이터 타입을 기본적으로 지원하지 않기 때문입니다. 이제 저는 test를 통과시키기 위해 test container를 적용하여 mysql 환경에서 테스트가 동작하도록 변경하면 됩니다.

```groovy
testImplementation 'org.testcontainers:junit-jupiter:1.19.0'
testImplementation 'org.testcontainers:mysql'
```

그리고 테스트를 성공시켰습니다.

하지만 롤백하였습니다. 이유는 
1. 잘 작동하는 서비스를 클러스터링을 위해 **너무나 많은 변경**이 생겼습니다
2. mysql 공간 데이터 타입의 내용이 너무 방대하여 팀원 모두가 알기 어렵고, 따라서 **에러가 발생했을 때 해결**하기가 힘들다
3. **공간 데이터 타입을 지원하지 않는 데이터베이스가 있다.** 지원하더라도 조금씩 문법이 다르다
4. 공간 데이터를 쓰지 않더라도 할 수 있는 **좋은 방법**이 떠올랐다

## Java로 클러스터링 하기

결국 클러스터링이라는 것은 여러 개의 마커를 합쳐 충전소의 개수만 보여주면 됩니다.
그러기 위해서는 영역을 나눠야합니다. 영역을 나누는 방법은 크게 두 가지가 있을 것 같습니다. 의미 있는 범위로 나누기, 격자로 나누기. 

물론 의미 있게 나누는 것이 개발하는 입장에서도 편할 것입니다. 하지만 저희 서비스가 배달이나, 쇼핑과 같이 **주소의 경계가 중요하지 않고**, 해당 위치 근처에 어떤 충전소가 있는지, 몇개가 있는지가 더 중요하다 판단하였습니다. 의미 있게 나눌 필요가 없습니다.

따라서 대한민국의 최극단을 기준으로 `2km X 2km` Grid를 생성했습니다.
```java
public class Grid {
    
    private Point top;
    private Point bottom;
    private List<StationPoint> stationPoints;
}

public class GridGenerator {

  public List<Grid> createKorea() {
    Point top = new Point(Latitude.from(TOP_LATITUDE), Longitude.from(TOP_LONGITUDE));
    Point bottom = new Point(Latitude.from(BOTTOM_LATITUDE), Longitude.from(BOTTOM_LONGITUDE));
    return create(top, bottom, LATITUDE_DIVISION_SIZE, LONGITUDE_DIVISION_SIZE);
  }
}

```
간단하게 설명드리면, 대한민국의 북서쪽의 극점과, 동남쪽 극점을 기준으로 사각형을 만들어, 2X2 의 Grid를 생성합니다. 대한민국을 나눴습니다. 이제 나눈 Grid에 포함되는 충전소를 찾아서 넣어주면 초기화가 끝이 납니다.

```java

public class StationGridService {

    public List<Grid> assignStationGrids(List<Grid> grids, List<StationPoint> stations) {
        for (Grid grid : grids) {
            for (StationPoint station : stations) {
              Point point = Point.of(station.latitude(), station.longitude());
              if (grid.isContain(point)) {
                grid.addPoint(point);
              }        
            }
        }
        return grids;
    }
}
```
먼저 찾아온 충전소의 좌표가 2X2 Grid에 포함된다면 추가해주는 메서드입니다.

```java
public class StationGridFacadeService {
    public List<GridWithCount> createGridWithCounts() {
        GridGenerator gridGenerator = new GridGenerator();
        List<Grid> grids = gridGenerator.createKorea();
        int size = 1000;
        int page = 0;
        while (size == 1000) {
            log.info("page : {}", page);
            List<StationPoint> stationPoint = stationQueryService.findStationPoint(page, size);
            stationGridService.assignStationGrids(grids, stationPoint);
            page++;
            size = stationPoint.size();
        }
        List<GridWithCount> list = grids.stream()
                .filter(Grid::hasStation)
                .map(it -> GridWithCount.createCenterPoint(it, it.stationSize()))
                .toList();
        return new ArrayList<>(list);
    }
}
```
이제 데이터베이스에 있는 충전소를 적당한 사이즈로 조금씩 가져와 아까 만들었던 메서드로 Grid에 추가해줍니다.

이 정보들을 데이터베이스에 저장하여 관리하려 했으나, 굳이 데이터베이스에 저장할 필요는 없습니다. 오히려 데이터베이스에 저장하는 것이 손해라고 생각했습니다. 이유는 충전소가 추가되었을 때, 해당 위치가 포함되는 Grid를 찾으려면 `findAll()` 메서드로 모든 Grid를 가져와야하고, 그리고 해당하는 부분에 넣어줘야합니다. 


하지만 메모리에서 관리한다면 충전소가 추가되었을 때 event나 다른 메서드를 통해 이미 들고있는 메모리에 로드되어있는 Grid에 count만 추가하면 되기 때문에 훨씬 효율적입니다.

따라서 [저번 게시글](https://drunkenhw.github.io/java/local-caching/)에서 설명했듯 메모리에 캐싱하고 이분 탐색을 사용하여 범위 조회를 하도록 만들었습니다.

다음은 2X2 Grid로 만들더라도, 아주 축소가 된다면 마커들이 잔뜩 보여 처음 얘기했던 문제가 또 발생할 것입니다. 그래서 동적으로 화면을 나눠 그곳에 해당하는 충전소 개수를 추가하는 기능을 만들 것입니다. 

아래는 화면의 중심점과 가로 세로 길이를 받아 사각형의 끝 점을 구하고, 클라이언트가 원하는 사이즈로 나눠 Grid를 생성해주는 메서드 입니다.
```java
public class StationGridFacadeService {

    private List<Grid> createDisplayGrid(ClusterRequest request) {
        GridGenerator gridGenerator = new GridGenerator();
        Coordinate coordinate = Coordinate.of(request.latitude(), request.latitudeDelta(), request.longitude(), request.longitudeDelta());
        BigDecimal maxLatitude = coordinate.maxLatitudeValue();
        BigDecimal minLongitude = coordinate.minLongitudeValue();
        BigDecimal minLatitude = coordinate.minLatitudeValue();
        BigDecimal maxLongitude = coordinate.maxLongitudeValue();
        return gridGenerator.create(Point.of(maxLatitude, minLongitude), Point.of(minLatitude, maxLongitude), request.latitudeDivisionSize(), request.longitudeDivisionSize());
    }
}
```

이렇게 생성한다면 무조건 2X2가 아닌, 동적으로 화면에 따라 3X3일수도 있고 100X100 로 생성할 수 있습니다. 

이제 마지막으로 생성한 Grid에다 충전소의 개수를 추가하듯 메모리에 캐싱해놓은 2X2 그리드를 추가하면 끝입니다.
```java
public class StationGridFacadeService {
    public List<GridWithCount> counts(ClusterRequest request) {
        List<GridWithCount> gridWithCounts = stationGridCacheService.findGrids(request);
        List<Grid> displayGrid = createDisplayGrid(request);
        List<Grid> grids = stationGridService.assignStationGridsWithCount(displayGrid, gridWithCounts);
        List<GridWithCount> list = grids.stream()
                .filter(Grid::existsCount)
                .map(it -> GridWithCount.createCenterPoint(it, it.getCount()))
                .toList();
        return new ArrayList<>(list);
    }
}
```
이렇게 다른 라이브러리 없이 순수 자바를 이용하여 클러스터링 기능을 완성했습니다.
![final](https://github.com/woowacourse-teams/2023-car-ffeine/assets/106640954/c65d9a51-5d3d-407b-9d72-13f06580a502)


처음 2400ms 걸리던 화면의 요청과 같은 요청을 클러스터링을 이용하였을 때는 171ms가 걸리게 되었습니다. 같은 조건은 아니지만은 약 **14배** 빨라졌습니다.

![final-time](https://github.com/woowacourse-teams/2023-car-ffeine/assets/106640954/b39a16fa-0d19-43d9-971e-07c7f7cf5b43
)

마지막으로 지금은 Grid의 중심 좌표만을 찍어주고 있지만, 추후에는 K-Means 알고리즘을 도입하여 좀 더 신뢰성있는 정보를 제공하려고 합니다.

## 결론
1. java 만으로도 해결할 수 있는 문제가 많은 것 같습니다.
2. java 재밌습니다