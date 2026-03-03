---
title: 数据库表主键 BIGINT vs VARCHAR 选型
categories: [工作总结]
tags: [数据库, 主键, 雪花算法]
---

在分布式系统中，雪花算法（Snowflake）是生成全局唯一 ID 的主流方案。但在数据库建表和 Java 代码实现时经常面临一个选择：**ID 到底该用 `BIGINT` (Long) 还是 `VARCHAR` (String)？**

## 1. 核心矛盾：性能 vs 兼容性

这个问题的本质，是**数据库存储性能**与**前端语言兼容性**之间的博弈。

### 方案 A：使用 BIGINT (Long)

这是数据库层面的最优解，但对前端（JavaScript）不友好。

- **MySQL**: `BIGINT(20)`
- **Java**: `java.lang.Long`

### 方案 B：使用 VARCHAR (String)

这是为了迁就前端的“偷懒”解法，牺牲了数据库性能。

- **MySQL**: `VARCHAR(20)`
- **Java**: `java.lang.String`

---

## 2. 为什么数据库层面 BIGINT 完胜？

从 DBA 和后端架构的角度来看，**必须使用 BIGINT**，原因如下：

1. **存储空间 (Storage)**
    - **BIGINT**: 固定占用 **8 字节**。
    - **VARCHAR**: 雪花 ID 为 19 位，存储需要 **19 字节 + 1 字节长度标识 = 20 字节**。
    - **结论**: String 的空间消耗是 Long 的 **2.5 倍**。这不仅浪费磁盘，更浪费宝贵的内存（Buffer Pool 能缓存的索引页变少）。
2. **索引效率 (Index Performance)**
    - **BIGINT**: CPU 处理整数比较极快，且因为体积小，B+ 树的层级更低，磁盘 I/O 更少。
    - **VARCHAR**: 字符串比对慢，且索引占用空间大，导致查询效率下降。

---

## 3. 为什么很多项目在用 String？(JS 精度大坑)

既然 `BIGINT` 这么好，为什么很多代码仓里还是用的 `String`？

原因是 **JavaScript 的精度丢失问题**。

- **Java `Long` 最大值**: $2^{63}-1$ (约 $9.22 \times 10^{18}$)
- **JS `Number` 最大安全值**: $2^{53}-1$ (约 $9.00 \times 10^{15}$)

雪花算法生成的 ID 通常在 19 位左右，远超 JS 的安全范围。
如果直接透传 `Long` 给前端，**最后几位会被四舍五入或变成 0**，导致前端回传 ID 时找不到数据。

**错误示例**:

```json
// 后端返回
{ "id": 1234567890123456789 }

// 前端接收变成
{ "id": 1234567890123456800 }
```

## 4. 最佳实践：各司其职

**不要为了前端的限制而牺牲数据库的性能。**

完美的解决方案是：**后端存 Long，前端收 String。**

### 架构设计

1. **MySQL**: 使用 BIGINT(20)，保证存储和索引性能。
2. **Java Entity**: 使用 Long，保证计算效率。
3. **JSON 序列化**: 在返回给前端时，自动将 Long 转为 String。

### 代码实现 (Spring Boot / Jackson)

我们只需要在 Entity 类的 ID 字段上添加一个注解，即可解决所有问题：

```java
import com.fasterxml.jackson.databind.annotation.JsonSerialize;
import com.fasterxml.jackson.databind.ser.std.ToStringSerializer;

public class User {

    /**
     * 数据库类型: BIGINT
     * Java 类型: Long
     * 
     * @JsonSerialize: 告诉 Jackson 在序列化成 JSON 时，
     * 使用 ToStringSerializer 将其转为 String 返回给前端。
     */
    @JsonSerialize(using = ToStringSerializer.class)
    private Long id;

    // 其他字段...
}
```

或者在全局配置 (WebMvcConfigurer) 中统一处理，让所有 Long 类型在序列化时都转为 String。

