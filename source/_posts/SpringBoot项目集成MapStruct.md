---
title: SpringBoot项目集成MapStruct
date: 2026-02-10 09:29:22
tags: MapStruct
categories: Springboot
---

官网：https://mapstruct.org/

github地址：https://github.com/mapstruct/mapstruct

# 1. MapStruct是什么？

MapStruct 是一个代码生成器，它基于约定优于配置的方法极大地简化了 Java bean 类型之间映射的实现。生成的映射代码使用简单的方法调用，因此速度快、类型安全且易于理解。

多层应用程序通常需要在不同的对象模型（例如实体和 DTO）之间进行映射。 编写这样的映射代码是一项乏味且容易出错的任务。 MapStruct 旨在通过尽可能自动化来简化这项工作。

# 2. MapStruct 和BeanUtils.copyProperties 对比

| 维度       | MapStruct              | BeanUtils.copyProperties |
| ---------- | ---------------------- | ------------------------ |
| 工作方式   | **编译期生成代码**     | **运行期反射**           |
| 类型安全   | ✅ 编译期校验           | ❌ 无校验                 |
| 性能       | 🚀 极快（普通方法调用） | 🐢 慢（反射 + 类型判断）  |
| 出错时机   | **编译时报错**         | **运行时才暴雷**         |
| 字段不一致 | ❌ 编译失败             | 😅 静默忽略               |
| 可维护性   | ✅ 高                   | ❌ 低                     |
| IDE 可读性 | ✅ 生成代码可看         | ❌ 黑盒                   |
| 适合规模   | 中大型项目             | 小脚本 / Demo            |

# 3. 与Springboot整合

## 3.1. 基础依赖引入

```xml
...
<properties>
  <org.mapstruct.version>1.6.3</org.mapstruct.version>
</properties>
...
<dependencies>
  <dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>${org.mapstruct.version}</version>
  </dependency>
</dependencies>
...
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.13.0</version>
      <configuration>
        <source>17</source>
        <target>17</target>
        <annotationProcessorPaths>
          <path>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct-processor</artifactId>
            <version>${org.mapstruct.version}</version>
          </path>
        </annotationProcessorPaths>
      </configuration>
    </plugin>
  </plugins>
</build>
...
```

## 3.2. MapStruct 可与 Project Lombok 一起使用

https://mapstruct.org/documentation/stable/reference/html/#lombok

Lombok 1.18.16 引入了重大更改（更改日志）。 必须添加附加注释处理器 lombok-mapstruct-binding (Maven)，否则 MapStruct 将停止与 Lombok 配合使用。 这解决了 Lombok 和 MapStruct 模块的编译问题。

```xml
	<properties>
		<java.version>17</java.version>
        <org.mapstruct.version>1.6.3</org.mapstruct.version>
        <lombok.version>1.18.32</lombok.version>
	</properties>
...
   <dependency>
      <groupId>org.mapstruct</groupId>
      <artifactId>mapstruct</artifactId>
      <version>${org.mapstruct.version}</version>
  </dependency>
  <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <version>${lombok.version}</version>
      <scope>provided</scope>
  </dependency>
...
  <configuration>
      <source>17</source>
      <target>17</target>
      <annotationProcessorPaths>
          <path>
              <groupId>org.mapstruct</groupId>
              <artifactId>mapstruct-processor</artifactId>
              <version>${org.mapstruct.version}</version>
          </path>
          <path>
              <groupId>org.projectlombok</groupId>
              <artifactId>lombok</artifactId>
              <version>${lombok.version}</version>
          </path>
          <path>
              <groupId>org.projectlombok</groupId>
              <artifactId>lombok-mapstruct-binding</artifactId>
              <version>0.2.0</version>
           </path>
      </annotationProcessorPaths>
  </configuration>
```

# 4. 使用方式

## 4.1.   单源对象映射   

### 4.1.1. 需要转换的两个实体

```java
@NoArgsConstructor
@AllArgsConstructor
@Data
public class Car {
    private String make;
    private int numberOfSeats;
    private String type;
}
@NoArgsConstructor
@AllArgsConstructor
@Data
public class CarVo {
    private String make;
    private int seatCount;
    private String type;
}
```

### 4.1.2. mapper interface

```java
@Mapper(componentModel = "spring")
public interface CarConverter {

    // 定义不同字段的映射关系 numberOfSeats 和 seatCount
    @Mapping(source = "numberOfSeats", target = "seatCount")
    CarVo toVo(Car car);
}
```

### 4.1.3. 测试使用

```java
@SpringBootTest
public class TestConvert {

    @Autowired
    private CarConverter carConverter;

    @Test
    public void shouldMapCarToDto() {
        Car car = new Car( "Morris", 5, "客车" );
        CarVo carVo = carConverter.toVo( car );

        assertThat(carVo).isNotNull();
        assertThat( carVo.getMake() ).isEqualTo( "Morris" );
        assertThat( carVo.getSeatCount() ).isEqualTo( 5 );
        assertThat( carVo.getType() ).isEqualTo( "客车" );
    }
}
```

## 4.2.  多源聚合映射  

**需求场景：**

- 原实体类：`Car` + `Owner`
- 目标实体类：`CarInfoVO`
- 需求：
- `Car.make`→ `CarInfoVO.make`
- `Car.make`→ `CarInfoVO.make`
- `Owner.name` → `CarInfoVO.ownerName`
- `Owner.age` → `CarInfoVO.ownerAge`
- 如果原实体有同名属性（如 `id`），指定 `Car.id` 优先

### 4.2.1. 源实体

```java
@NoArgsConstructor
@AllArgsConstructor
@Data
public class Car {
    private Long id;
    private String make;
    private int numberOfSeats;
    private String make;
}
@Data
public class Owner {
    private Long id;
    private String name;
    private Integer age;
}
```

### 4.2.2. 目标实体

```java
@NoArgsConstructor
@AllArgsConstructor
@Data
public class CarVo {
    private String make;
    private int seatCount;
    private String type;
}
```

### 4.2.3. mapper interface

```java
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.Mappings;

@Mapper(componentModel = "spring")
public interface CarInfoConverter {

    // 将 Car 和 Owner 属性合并到同一个 VO
    @Mappings({
        @Mapping(source = "car.id", target = "id"),       // 同名属性，Car 优先
        @Mapping(source = "owner.name", target = "ownerName"), // 和最终Vo名字不一致，需要写出来，一致则可以省略
        @Mapping(source = "owner.age", target = "ownerAge")
    })
    CarInfoVO toCarInfoVO(Car car, Owner owner);
}
```

### 4.2.4. 测试使用

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.assertj.core.api.AssertionsForClassTypes.assertThat;

@SpringBootTest
public class TestConvert {

    @Autowired
    private CarInfoConverter carInfoConverter;

    @Test
    void testMergeCarAndOwnerToVO() {
        Car car = new Car();
        car.setId(1L);
        car.setMake("BMW");
        car.setType("客车");

        Owner owner = new Owner();
        owner.setId(99L); // 同名属性，VO使用Car的id
        owner.setName("Alice");
        owner.setAge(30);

        CarInfoVO vo = carInfoConverter.toCarInfoVO(car, owner);

        assertThat(vo.getId()).isEqualTo(1L);        // Car.id 优先
        assertThat(vo.getMake()).isEqualTo("BMW");
        assertThat(vo.getType()).isEqualTo("客车");
        assertThat(vo.getOwnerName()).isEqualTo("Alice");
        assertThat(vo.getOwnerAge()).isEqualTo(30);
    }
}
```

# 5. IDEA 相关插件

![img](https://cdn.nlark.com/yuque/0/2026/png/29413899/1770619454356-c8f71a8d-d5ea-4d5c-bb4c-5672cf8a6b2c.png)