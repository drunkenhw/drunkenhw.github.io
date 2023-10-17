---
title:  "Domain vs Entity"
excerpt: "Domain과 Entity의 차이"

categories:
  - java
tags:
  - java
last_modified_at: 2023-05-19T00:29:00-05:00
---

# Domain vs Entity

먼저 이 글을 쓰는 이유는 많이 사용하는 용어이지만, 정확히 용어를 정리하는 것이 어려워 내가 정리한 생각을 적어보려고 한다.

### Domain

도메인 객체는 애플리케이션의 **핵심 비즈니스 로직을 모델링**하는 객체다. 도메인은 데이터베이스 설계 혹은 UI 와 같은 다른 부분의 영향을 받지 않고 순전히 비즈니스 로직을 수행하는데 목적을 둔다. 이렇게 함으로써 비즈니스 로직의 복잡도를 추상화하고, 코드의 유지보수성을 향상 시킬 수 있다. 그리고 Domain은 POJO(Plain Old Java Object)에 가까울수록 좋다. 다른 프레임워크, 라이브러리의 의존을 피할 수 있다면 사용하지 않는 것이 최선이다.

### Entity
*(Entity라는 개념은 추상적인데, 아래의 설명은 영속성 계층의 Entity라고 가정한다)*

보통 **데이터베이스의 테이블과 매핑되는 클래스**이다. 테이블과 매핑을 함으로써 데이터베이스에 저장되어 있는 정보를 객체로 바꿀 수 있는 것이다. Entity는 식별자(ID)와 같은 기본 키, 테이블의 각 속성을 모델링하는 멤버 변수, 그리고 데이터베이스와 상호작용하는 메서드를 포함한다. 

### DAO

DAO는 Data Access Object의 준말로, 직역하면 **데이터에 접근하는 객체**다. 여러 데이터베이스와 통신을 하는 객체다. 각 밴더 회사의 API를 이용해 데이터베이스에 접근하여 CRUD의 작업을 수행한다. 

### Repository

Repository는 DDD(Domain Driven Develop)에서 나온 개념이다. 쉽게 표현하면 Repository는 **Domain객체의 컬렉션**이라고 생각하면 된다. Repository는 비즈니스 로직에 따라 Domain 객체의 상태를 관리하는 저장소이다. 하지만 이 Domain의 정보가 Repository가 데이터베이스에 저장된다는 보장은 없다. 그냥 비즈니스 로직을 수행하기 위해 객체의 상태를 저장해놓는 곳이라는 뜻이다. 

---

그럼 이 개념을 비교해보자. 

### Domain vs Entity

먼저 도메인과 엔티티의 차이를 정리해보자, 도메인은 비즈니스 로직을 수행하는 객체다. 엔티티는 도메인의 상태를 데이터베이스에 저장하는 객체다. 

그럼 무엇이 다른가, 엔티티에는 비즈니스 로직이 없다. 하지만 엔티티에는 객체의 상태를 저장하고 상태를 알기위한 식별자 (ID)가 존재한다. 

하지만 도메인은 상태가 없는가? 비즈니스 로직을 수행하기 위해 도메인에도 상태가 분명 필요한 경우가 있다. 그럼 그 상태는 어떻게 비교할 것인가. 아래와 같은 간단한 Car 클래스가 있다.
```java
public class Car {

    private final String name;

    public Car(String name) {
        this.name = name;
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (o == null || getClass() != o.getClass()) {
            return false;
        }
        Car car = (Car) o;
        return name == car.name;
    }

    @Override
    public int hashCode() {
        return Objects.hash(name);
    }
}
```
여기서 자동차는 이름이 같으면 같은 자동차라고 정의되어 있다. 하지만 이름이 같다고 모두 같은 자동차일까? 

나의 박스터와 다른 사람의 박스터가 이름이 같다고 공유된다면 이상하다. 그래서 이 상태를 저장하는 도메인에는 식별자가 필요할 것이고, 그 식별자를 통해 동등성을 비교해야할 것이다. 

그리고 아래와 같이 수정할 수 있다.


```java
public class Car {

    private final Long id;
    private final String name;
    
    ...
  
    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (o == null || getClass() != o.getClass()) {
            return false;
        }
        Car car = (Car) o;
        return id == car.id;
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}
```
 이제서야 내 차와 다른 사람의 차가 정확히 구분된다. 
 
정리하자면, Entity는 무언가의 상태를 저장하기 위한 개념이고 Domain은 비즈니스 로직을 수행하는 객체인 것이다. 

---

### 결론

**Domain과 Entity의 차이를 두고 구분을 할 필요가 없다.**

Domain도 Entity라 할 수 있다. 그래서 Domain 계층의 Entity와 영속성 계층의 Entity로 나눌 수 있다.   
**(물론 꼭 나눌 필요는 없다.)**  
하지만 Domain 객체 그대로 데이터베이스에 저장하기 힘든 경우가 있기 때문에 Entity라는 객체를 만들어 테이블과 매핑되게 만들어 상태를 저장할 수 있다.



잘못된 내용에 대한 피드백 부탁드립니다.