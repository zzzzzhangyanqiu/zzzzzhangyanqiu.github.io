---
title: Spring 事务的可重复读，为什么会让你的数据写两次？
date: 2026-03-25
tags:
  - Java
  - Spring
  - 事务
categories:
 - Java
---

线上出现了两条一模一样的订单记录，代码看了三遍没问题，单测也能过，偏偏在压测环境下稳定复现。

如果你也碰到过类似的情况，大概率就是这个问题：**Spring 事务在可重复读隔离级别下，"读到的世界"和"真实的世界"对不上了。**

---

## 先把问题说清楚

MySQL InnoDB 默认的事务隔离级别是可重复读（REPEATABLE READ），Spring 也继承了这个默认值。

可重复读的意思是：**一个事务启动后，不管外面发生了什么，它每次读同一行数据的结果都是一样的。** 听起来很合理，甚至是个好特性。

但问题就藏在这里。

这个"一致性"是通过 MVCC（多版本并发控制）实现的——事务启动时会生成一个快照，后续所有读操作都基于这个快照。你读到的，是事务开始那一刻的世界，不是现在的世界。

把这个机制和"先读后写"的业务逻辑放在一起，麻烦就来了。

---

## 三种典型场景

### 场景一：并发插入 —— 明明检查过不存在，还是写进去了重复数据

这是最常见的一种，电商、社交平台都踩过。

业务逻辑很简单：注册时先查用户名有没有被占用，没有就插入。

```java
@Transactional
public void register(String username) {
    User user = userMapper.selectByUsername(username);
    if (user == null) {
        userMapper.insert(new User(username));
    }
}
```

单个请求跑完全没问题。但两个请求同时进来：

```
时间线：
T1  事务A：SELECT username='tom' → 查不到
T2  事务B：SELECT username='tom' → 查不到
T3  事务B：INSERT → 成功，提交
T4  事务A：INSERT → 也成功了！（因为快照里还是查不到那条记录）
结果：数据库里出现两个 tom
```

事务A的快照在 T1 就生成了，T3 时事务B插入的那条记录对事务A是不可见的。事务A一路走到 T4，完全不知道自己在做一件已经有人做过的事。

这就好像两个人同时去超市买了最后一瓶酱油，都觉得货架上还有一瓶，都放进了购物车。

**解法：给 SELECT 加锁，或者数据库加唯一索引（最好两个都加）。**

```java
@Transactional
public void register(String username) {
    // SELECT ... FOR UPDATE，让第二个事务在这里等待
    User user = userMapper.selectByUsernameForUpdate(username);
    if (user == null) {
        userMapper.insert(new User(username));
    }
}
```

唯一索引是最后一道防线，即使业务代码有漏洞，数据库也会在 INSERT 时抛出异常阻止重复数据落库。两者配合，才算稳。

---

### 场景二：更新丢失 —— 改了，但其实没改成

这个场景更隐蔽，因为代码执行完没有报错，数据也确实更新了，只是更新的值不对。

两个操作同时修改账户余额：

```java
@Transactional
public void addBalance(Long userId, int amount) {
    User user = userMapper.selectById(userId);     // 读到 balance=100
    user.setBalance(user.getBalance() + amount);   // 算出 100+50=150
    userMapper.updateById(user);                   // 写入 150
}
```

并发时序：

```
T1  事务A：读到 balance=100
T2  事务B：读到 balance=100
T3  事务B：写入 balance=100+200=300，提交
T4  事务A：写入 balance=100+50=150，提交（覆盖了事务B的结果）
最终：balance=150，事务B的那次 +200 消失了
```

事务A的快照里 balance 始终是 100，所以它算出来的结果是基于一个已经过时的值。事务B写进去的 300 被直接覆盖，没有任何报错。

余额少了 200，日志里查不到原因，这种 bug 最难排查。

**解法：用乐观锁，加版本号。**

```java
// 表里加一个 version 字段
@Transactional
public void addBalance(Long userId, int amount) {
    User user = userMapper.selectById(userId);
    int affected = userMapper.updateWithVersion(
        userId,
        user.getBalance() + amount,
        user.getVersion()   // 更新时带上版本号
    );
    if (affected == 0) {
        throw new RuntimeException("并发冲突，请重试");
    }
}
```

对应的 SQL：

```sql
UPDATE user 
SET balance = #{newBalance}, version = version + 1
WHERE id = #{id} AND version = #{version}
```

如果版本号对不上，`affected=0`，事务重试或者报错，不会静默地写入错误数据。

乐观锁的思路是：不拦截读，只在写的时候验证"你读到的那个版本还是最新的吗"。适合读多写少、冲突概率不高的场景。

---

### 场景三：同一事务内写了两次 —— 自己把自己的数据覆盖了

前两个场景都是并发引起的，这个场景更特殊：单线程，单事务，照样出问题。

```java
@Transactional
public void processOrder(Long orderId) {
    Order order = orderMapper.selectById(orderId);

    // 第一次更新：标记为处理中
    order.setStatus(OrderStatus.PROCESSING);
    order.setUpdateTime(LocalDateTime.now());
    orderMapper.updateById(order);  // UPDATE #1

    // ... 一些业务逻辑 ...

    // 第二次更新：标记为完成
    order.setStatus(OrderStatus.DONE);
    orderMapper.updateById(order);  // UPDATE #2
    // 问题：updateTime 还是第一次设置的那个时间，没有更新
}
```

这里有个细节问题：第一次 `updateById` 执行后，数据库里 `update_time` 已经是新时间了，但内存里的 `order` 对象还是同一个引用。第二次调用 `updateById` 时，如果 ORM 框架（比如 MyBatis-Plus）把整个对象字段都写进 SQL，就会把第一次更新进去的 `update_time` 又覆盖回去。

更严重的版本是：事务内对同一个对象做了两次语义不同的修改，但因为对象引用没变，第二次 `updateById` 把第一次的部分字段改回去了。

**解法：拆开来写，或者明确每次只更新必要的字段。**

```java
@Transactional
public void processOrder(Long orderId) {
    // 直接用精确的 SQL，不要把整个对象塞进去
    orderMapper.updateStatus(orderId, OrderStatus.PROCESSING, LocalDateTime.now());

    // ... 业务逻辑 ...

    orderMapper.updateStatus(orderId, OrderStatus.DONE, LocalDateTime.now());
}
```

或者用 MyBatis-Plus 的 `UpdateWrapper` 明确指定要更新哪些字段，而不是 `updateById`（全字段覆盖）。

---

## 几个常被混淆的点

**Q：加了 `@Transactional` 就会有这个问题吗？**

不一定。单线程、无并发的场景里，可重复读只是让你的事务内读一致，没有副作用。问题出在并发写场景，或者事务内对同一对象多次写但逻辑没处理好的情况。

**Q：把隔离级别改成 `READ_COMMITTED` 能解决吗？**

能解决快照读导致的并发插入问题，因为 READ COMMITTED 下每次读都是当前最新数据。但更新丢失的问题依然存在，而且引入了不可重复读，反而可能带来新的问题。隔离级别往上调（SERIALIZABLE）性能代价很大，一般不推荐。

**Q：`SELECT FOR UPDATE` 和乐观锁怎么选？**

- 写冲突频繁、容忍不了失败重试 → `SELECT FOR UPDATE`（悲观锁）
- 写冲突少、能接受失败重试 → 乐观锁（版本号）
- 两者都加上数据库唯一索引 → 最稳

---

## 总结

回到最开始的问题：可重复读本身不是 bug，它是为了解决不可重复读问题而设计的。但它带来的副作用是，事务内的读操作活在一个"冻结的快照"里，而写操作却实实在在地发生在当下。

当你的业务逻辑依赖"读的结果来决定写什么"的时候，这个时间差就会造成问题：

- 并发插入：快照里没有，现实里已经有了 → 重复数据
- 并发更新：快照里是旧值，现实里已经改了 → 更新丢失
- 同事务多次写：对象引用没变，字段值被旧数据覆盖 → 数据回退

解法不复杂，核心思路是：**把"读"和"写"绑定起来，不给它们中间留空隙。** 悲观锁是直接把这段时间锁住，乐观锁是在写的时候验证这段时间没人动过，唯一索引是最后的兜底。

三种手段根据场景搭配使用，比单独依赖任何一种都可靠。
