---
title: MapStruct를 이용한 DTO 매핑 자동화(feat.관심사의 분리)
description: >-
  MapStruct를 이용한 DTO 매핑 자동화를 이용해서 우리 한 번 신나게 달려가봐요!
author: author_js
date: 2025-03-24 17:55:00 +0900
categories: [JAVA, Spring Boot]
tags: [java, spring, boot, 자바, 스프링부트, 스프링, JPA, entity, mapStruct, dto]
pin: true
---

개발을 하다 보면 '관심사의 분리(Separation of Concerns)'라는 말을 자주 접하게 됩니다. 한 클래스나 컴포넌트는 가능한 하나의 책임만을 담당하도록 설계하는 것이 중요하다는 개념인데요. 프론트엔드 개발에서도 하나의 컴포넌트가 단 하나의 기능만 담당하도록 권장되고, 백엔드 역시 마찬가지 입니다.

백엔드, 특히 JPA를 사용할 때 Entity는 도메인의 핵심 로직과 데이터 영속성 관리라는 역할에만 집중하는 것이 좋습니다. 때문에 실제 데이터를 주고받기 위해서는 Entity가 아닌 별도의 DTO(Data Transfer Object)를 만들어 사용하는 경우가 많습니다.

## DTO로 변환하는 귀찮음

저 역시 처음에는 별도의 Mapper 클래스를 만들어서 Entity를 DTO로 변환하는 정적 메서드를 작성했습니다. 이렇게 하면 구조적으로는 깔끔하지만 Entity마다 매핑을 위한 클래스를 하나씩 만드는 것이 꽤 번거롭고 귀찮은 일이 아닐 수 없습니다.

```java
public class EntityMapper {

  public static EntityDto toDto(Entity entity) {
    return EntityDto.builder()
        .id(entity.getId())
        .name(entity.getName())
        .build();
  }
}
```

Entity와 DTO가 많아질수록 이 작업이 상당히 부담스러워졌습니다.

## 자동화 신세계 - MapStruct

하지만 사람은 항상 배우면서 발전하는 존재라는 것을 또 한 번 느끼게 됩니다. 이런 귀찮음을 덜어줄 수 있는 놀라운 라이브러리를 알게 되었습니다. 바로 MapStruct입니다.

MapStruct는 interface만 정의해두면 필드 이름이 같은 Entity와 DTO를 자동으로 매핑해주는 강력한 라이브러리입니다. 물론 복잡한 로직이 들어가면 일부 수동으로 커스터마이징해야 하겠지만, 일반적으로 단순히 이름만 같은 필드들을 자동으로 매핑해주기 때문에 엄청난 시간을 절약할 수 있습니다.

## MapStruct 사용방법 간단 예시

아래는 MapStruct를 이용한 간단한 사용 예제 입니다.

### 1. Dependency 추가하기

Maven이라면 다음과 같이 설정합니다.
gradle은 찾아서!

```java
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>1.5.5.Final</version>
</dependency>
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct-processor</artifactId>
    <version>1.5.5.Final</version>
    <scope>provided</scope>
</dependency>
```

### 2. Interface 정의하기

```java
@Mapper(componentModel = "spring")
public interface EntityMapper {
    EntityMapper INSTANCE = Mappers.getMapper(EntityMapper.class);

    EntityDto entityToDto(Entity entity);
}
```

두 개의 클래스가 필드가 같다면 위처럼 한 줄의 코드로 자동 매핑이 가능!

### 3. 사용 예

```java
Entity entity = new Entity();
entity.setId(1L);
entity.setName("Test");

EntityDto dto = EntityMapper.INSTANCE.entityToDto(entity);
```

이렇게 하면 Entity의 필드들이 자동으로 DTO의 필드로 옯겨집니다. 정말 깔끔하쥬?

## 결론!

관심사의 분리는 코드 유지보수에 있어 정말 중요한 원칙입니다. MapStruct 같은 라이브러리를 적극 활용하면, 이런 원칙을 지키면서도 코드 작성을 훨씬 효율적이고 편리하게 할 수 있습니다. ㅎㅎ
