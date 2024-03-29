---
title: 'JPA에서 ID가 있는 Entity에 대해 save 시에 select 쿼리가 나가는 이유'
excerpt: "그리고 그 문제를 해결하기"

categories:
  - jpa
tags:
  - jpa
last_modified_at: 2023-07-09T19:14:00-05:00
---

안녕하세요 박스터 입니다.

먼저 이번에 글을 쓰게된 계기를 말씀드리겠습니다. 저희 팀은 공공 데이터 API에서 받아온 충전소와, 충전기들의 ID를 그대로 사용하고 있습니다.
물론 다른 API, 제가 제어할 수 없는 곳에 의존하는 것은 좋지 않다고 생각합니다.

하지만 데이터를 받아오는 과정에서 마주한 성능적인 문제 때문에 그대로 사용하고 있습니다. 전국의 충전소는 6만개, 충전소 안에 존재하는 충전기는 23만기입니다.
하지만 공공 데이터는 충전소와, 충전기의 정보를 따로 제공하는 것이 아닌 중복된 충전소를 포함한 데이터를 충전기 개수만큼인 23만개의 row로 제공합니다.


따라서 저희가 ID를 따로 부여하게 된다면, 충전소를 저장하는 과정에서 받아오는 ID로 충전기를 연결해줘야하는데 그렇게 된다면 셀 수 없이 많은 쿼리가 발생합니다.

잠깐 생각해본다면
1. 충전소를 각각 저장하고 ID를 부여받는 쿼리 `6만번` (ID를 알아와야하기 때문에 batch를 사용할 수 없습니다.)
2. 충전소에서 받아온 ID를 충전기에 저장하는 쿼리 `최소 1번` (만약 batch로 23만건을 한번에 저장한다는 가정)

하지만 ID를 그대로 사용하게 된다면,
1. 충전소를 저장하는 쿼리 `최소 1번` (만약 batch로 6만건을 한번에 저장한다는 가정)
2. 충전기를 저장하는 쿼리 `최소 1번` (만약 batch로 23만건을 한번에 저장한다는 가정)

23만건이 넘는 정보를 확인했을 때, ID는 중복되지 않았고, 중복하지 않을 것이라 생각했습니다. 그 뿐만 아니라 처음 한번만 저장하는 것이 아닌 주기적으로 업데이트된 정보를
반영해주고 `update` or `save` 해주어야하기 때문에, ID를 그대로 가지고 있는 것이 훨씬 효율적이라 생각했습니다.

사족이 길었습니다. 각설하고 이런 방식으로 ID를 직접 넣어주는 경우 발생하는 문제에 대해 말씀드리겠습니다.

## ID를 직접 넣어준 Entity를 저장할 때

먼저 간단한 예제 Entity로 설명드리겠습니다.

```java
@Entity
public class ChargeStation {

    @Id
    private String stationId;

    private String stationName;

    ...
}

```
보통의 Entity와 다른 부분은 `@GeneratedValue(strategy = GenerationType.IDENTITY)` 이러한 ID 생성 전략에 대한 정보가 없습니다.

그리고 `save()` 코드를 호출하면 어떤 쿼리가 나가는지 확인해보겠습니다. 아래와 같이 아주 간단한 선릉역 충전소를 저장하는 테스트를 실행해보겠습니다.
```java
@DataJpaTest
class ChargeStationRepositoryTest {

    @Autowired
    private ChargeStationRepository chargeStationRepository;

    @Test
    void 충전소를_저장한다() {
        ChargeStation station = ChargeStationFixture.선릉역_충전소_충전기_2개_사용가능_1개;

        chargeStationRepository.save(station);

        ChargeStation expect = chargeStationRepository.findByStationId(station.getStationId()).get();
        assertThat(expect).isEqualTo(station);
    }
}
```

먼저 코드만 보면 먼저 `chargeStationRepository.save()` 호출과 함께 insert 쿼리 1번, 그리고 `chargeStationRepository.findByStationId()`에서 select 쿼리 1번
총 2번 발생할 것이라고 유추할 수 있습니다.

![query-three-times](https://github.com/car-ffeine/design-system/assets/106640954/f48b7f0f-3f39-41ce-8fcd-94b995e95fae)

하지만 예상과 다르게 위의 사진과 같이 쿼리가 총 3번 발생했습니다. 첫번째는 호출하지 않은 station id로 station을 조회하는 쿼리가 발생했습니다.

이유를 찾기 위해 `save()` 메서드를 디버깅 해봤습니다.

### save 시 SELECT 쿼리가 발생하는 이유


![save-method](https://github.com/car-ffeine/design-system/assets/106640954/b1db00b7-d7fb-4647-912c-6f8e2fe44974)
로직은 간단해보입니다. `isNew()` 를 통해 새로운 Entity인지 확인한 후, 새로운 Entity라면 `persist()`, 아니라면 `merge()`를 호출합니다.

여기서 `EntityManager#persist()` 메서드를 간단히 말씀드리면, 새로운 Entity를 영속화하는 메서드로 트랜잭션이 커밋될 때 데이터베이스에 저장합니다.

그리고 `EntityManager#merge()` 메서드는 준영속 상태의 Entity를 영속 상태로 변경하는데 사용합니다.
하지만 이때 영속성 컨텍스트에 존재하지 않는 객체라면 데이터베이스에서 조회 후 영속화하는 작업을 수행합니다.

`merge()`를 호출하기 때문에 SELECT 쿼리가 발생하고, 영속화하는 작업을 수행하는 것 입니다.

하지만 제가 저장한 객체는 확실히 새로운 Entity가 맞습니다. 하지만 `entityInformation.isNew()` 메서드는 false를 반환합니다.

그래서 어떤 것을 기준으로 새로운 Entity인 것을 구분하는지 알아보겠습니다.

### 새로운 Entity를 구분하는 기준

일단, 디버깅을 통해 isNew 메서드를 확인해보겠습니다.
![is-new](https://github.com/car-ffeine/design-system/assets/106640954/e4a56694-c623-46d8-badd-3345d557e29f)

간단합니다. 먼저 Entity에 ID를 가져옵니다. 그리고 id가 `primitive` 타입인지 확인 후, 아닐경우 id가 null 이면 새로운 Entity, 아닐경우 false를 반환합니다.

이때, `primitve` 타입이라면, id가 숫자인지 확인 후 id가 0이면 새로운 Entity, 아닐경우 false를 반환합니다.

## ID를 직접 넣어주는 객체는 JPA 사용을 포기해야할까?

결론부터 말씀드리면 아닙니다. 다른 방법으로 새로운 Entity 임을 증명할 수 있다면 `merge()`가 아닌 `persist()`를 호출하도록 만들 수 있습니다.

### 그럼 어떻게?
먼저 save() 메서드의 필드 중 `JpaEntityInformation`이라는 필드를 확인할 수 있습니다.

![entity-info](https://github.com/car-ffeine/design-system/assets/106640954/d9956fe6-07c7-41a9-9d6b-7c01b5f31c5d)

이 인터페이스는 Entity의 추가 정보를 알기 위해 필드에 있습니다.

해당 인터페이스의 구현체는 `JpaEntityInformationSupport`, `JpaMetamodelEntityInformation`, `JpaPersistableEntityInformation` 이렇게 3개의 클래스가 있습니다.

그럼 다른 방법으로도 `isNew()`가 구현되어 있을거라 추측을 할 수 있습니다. 디버깅을 통해 알아보겠습니다.

아까 위의 사진으로 보고 실제로 실행됐던 `isNew()` 메서드의 주인은 `JpaMetamodelEntityInformation` 클래스였습니다. 그래서 해당 클래스는 제외하고 다른 클래스를 보겠습니다.

먼저 `JpaPersistableEntityInformation` 클래스입니다.
![is-new-persistable](https://github.com/car-ffeine/design-system/assets/106640954/dc2293c3-2854-4619-9ef6-d08e55b4581b)
아주 간단하게 entity의 `isNew()`를 호출한다고 적혀있습니다. 하지만 `Persistable` 인터페이스를 구현한 `isNew()` 를 호출하는 것 입니다.

그럼 남은 하나의 클래스를 확인하겠습니다.

![info-support](https://github.com/car-ffeine/design-system/assets/106640954/f1d654c0-e741-4db7-8e7f-4e758c36133a)

위 사진처럼 이 클래스가 Entity 마다 `Persistable` 구현 유무에 따라 동적으로 구현체를 변경해주고 있었습니다.

그럼 답이 나온 것 같습니다. ID를 직접 할당하는 Entity에 `Persistable`을 구현해주면 됩니다.

### Persistable 구현하기
```java
@Entity
public class ChargeStation implements Pesistable{

    @Id
    private String stationId;

    private String stationName;

    @CreatedDate
    private LocalDateTime createdTime;

    ...

    @Override
    public Object getId() {
        return getStationId();
    }

    @Override
    public boolean isNew() {
        return createdTime == null;
    }
}
```

간단히 만들어봤습니다. `@CreatedDate`는 Entity가 처음 영속화될 때 동작하기 때문에 이 Entity의 CreateTime 필드가 null 이면 새로운 Entity라고 확신할 수 있습니다.
그럼 이렇게 인터페이스를 구현하고 아까 실행했던 테스트를 다시 실행해보겠습니다.

![solved](https://github.com/car-ffeine/design-system/assets/106640954/ea5db719-9919-42f4-b431-00e14d6fea5e)

깔끔하게 구현된 것을 확인할 수 있었습니다.

#### 결론
JPA는 많은 편의 기능을 제공해주는 것 같아보입니다. 쫄지맙시다.
