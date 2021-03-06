---
layout: post
title: "记tx-lcn分布式事务线上问题"
date: 2020-07-20 23:00
categories: "6.824"
tags:
    - distribution
---


最近几天生产爆出了个数据库查询的异常，线程是forkjoin连接池的，追到根源是从druid连接报出来的：

```java
2020-07-15 11:46:01.228  INFO [ForkJoinPool-1-worker-15]
......
Caused by: java.lang.NullPointerException: null
        at com.alibaba.druid.pool.DruidPooledConnection.transactionRecord(DruidPooledConnection.java:720)
        at com.alibaba.druid.pool.DruidPooledStatement.transactionRecord(DruidPooledStatement.java:284)
        at com.alibaba.druid.pool.DruidPooledPreparedStatement.execute(DruidPooledPreparedStatement.java:491)
```

查看这段源码：

```java
// DruidPooledConnection.java:720
710 protected void transactionRecord(String sql) throws SQLException {
720    if (transactionInfo == null && (!conn.getAutoCommit())) {
721        DruidAbstractDataSource dataSource = holder.getDataSource();
722        dataSource.incrementStartTransactionCount();
723        transactionInfo = new TransactionInfo(dataSource.createTransactionId());
724    }
...
```

也就是说conn是null，而这个conn是java.sql.Connection的实例，再往上追溯几行

```java
// DruidPooledPreparedStatement.java:491
486    public int getMaxFieldSize() throws SQLException {
487        checkOpen();
488
489        try {
490            return stmt.getMaxFieldSize();
491        } catch (Throwable t) {
492            throw checkException(t);
493        }
494    }
...
```

看checkOpen方法，没有抛异常，说明`closed==false`

```java
// DruidPooledPreparedStatement.java:491
protected void checkOpen() throws SQLException {
    if (closed) {
        Throwable disableError = null;
        if (this.conn != null) {
            disableError = this.conn.getDisableError();
        }

        if (disableError != null) {
            throw new SQLException("statement is closed", disableError);
        } else {
            throw new SQLException("statement is closed");
        }
    }
}
```

既然前期检查连接是正常的状态，那为什么之后`conn==null`，导致抛出异常。难道conn存在并发的可能，状态被其他线程修改了？

接着向上查找日志，这个数据库连接被分布式事务框架代理了：

```java
2020-07-15 11:46:01.215  DEBUG [ForkJoinPool-1-worker-15] com.codingapi.txlcn.tc.aspect.weave.DTXResourceWeaver - proxy a sql connection: com.codingapi.txlcn.tc.core.transaction.lcn.resource.LcnConnectionProxy@65907ef8.
```

如果有并发问题，难道这个连接被多线程使用，于是，接着查找关键字“65907ef8”，

```java
2020-07-15 11:46:00.476  DEBUG [http-nio-20790-exec-39] com.codingapi.txlcn.tc.aspect.weave.DTXResourceWeaver - proxy a sql connection: com.codingapi.txlcn.tc.core.transaction.lcn.resource.LcnConnectionProxy@65907ef8.
......

2020-07-15 11:46:01.227  WARN [http-nio-20790-exec-39] com.codingapi.txlcn.tc.core.transaction.lcn.resource.LcnConnectionProxy - transaction type[lcn] proxy connection:com.codingapi.txlcn.tc.core.transaction.lcn.resource.LcnConnectionProxy@65907ef8 closed.
```

对比时间可以发现，http-nio-20790-exec-39线程，处理应用端请求，开启了分布式的事务的代理连接，处理之后关闭了连接。在这中间，ForkJoinPool-1-worker-15的连接也被代理了，最后，在http的连接关闭后，forkjoin线程抛出了异常，那么，http线程处理过程中，为什么forkjoin线程也共用一个数据库连接？

forkjoin线程跑出异常的代码片段：

```java
// 并发执行检查
forkJoinConfig.getForkJoinPool().submit(() -> {
  list.parallelStream().forEach(checker -> {
    checker.check(param, result);
  });
}).join();
```
`getForkJoinPool()`获取了我们配置的forkjoin线程池，之后list的foreach中的操作就用的这个连接池处理的。

查看tx-lcn源码，数据库代理的部分：

```java
// com.codingapi.txlcn.tc.aspect.weave.DTXResourceWeaver
	public Object getConnection(ConnectionCallback connectionCallback) throws Throwable {
        DTXLocalContext dtxLocalContext = DTXLocalContext.cur();
        if (Objects.nonNull(dtxLocalContext) && dtxLocalContext.isProxy()) {
            String transactionType = dtxLocalContext.getTransactionType();
            TransactionResourceProxy resourceProxy = txLcnBeanHelper.loadTransactionResourceProxy(transactionType);
            Connection connection = resourceProxy.proxyConnection(connectionCallback);
            log.debug("proxy a sql connection: {}.", connection);
            return connection;
        }
        return connectionCallback.call();
    }
```

大概是获取事务的上下文环境，从而获取数据库连接的代理对象。

forkjoin的线程，日志打出了“proxy a sql connection”，说明`dtxLocalContext!=null`，那DTXLocalContext是个啥？

```java
// com.codingapi.txlcn.tc.core.DTXLocalContext
	/**
     * 获取当前线程变量。不推荐用此方法，会产生NullPointerException
     *
     * @return 当前线程变量
     */
    public static DTXLocalContext cur() {
        return currentLocal.get();
    }
///
    private final static ThreadLocal<DTXLocalContext> currentLocal = new InheritableThreadLocal<>();
```

那么问题应该在于currentLocal了，它是InheritableThreadLocal的实例。

回忆一下ThreadLocal的原理，每个线程都有一个map，通过map.get(threadlocal)，就能拿到当前线程存进去的对象了。

InheritableThreadLocal原理类似，只是一个子线程在初始化时，会复制父线程的map数据。

```java
// java.lang.Thread#init
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);

```
`inheritableThreadLocals`就是我们所说的线程的map，它是`ThreadLocal.ThreadLocalMap`的实例。

在我们的例子中，forkjoin的线程创建时，复制了父线程的数据，即父线程带有DTXLocalContext对象，
说明【ForkJoinPool-1-worker-15】线程是【http-nio-20790-exec-39】创建的，那就说明http线程执行中也用了同一个forkjoin线程池（找寻代码后，被证实）。

## 总结

分布式事务框架为了追踪请求Q1里的并发的情况，在InheritableThreadLocal对象中，存了分布式事务上下文，包括数据库连接，以便于之后子线程A也能拿到分布式事务上下文，获取到相同的数据库连接。

但是，如果子线程A在线程池中，且其他的请求Q2使用了这个线程池，且恰巧被分到了线程A，那么就获取到了相同的数据库连接，然而这个数据库连接由请求Q1控制，那么Q2就面临了数据库连接随时被关闭的风险。

相对的，如果请求Q2做了修改操作，那么也可能影响请求Q1的准确性，即对Q1来说，数据库连接被泄露。



## 思考，分布式事务上下文和线程池，如何共处？

从上面的案例，父线程在创建新线程时，隐式地将上下文信息拷贝给新线程，
但是，这个新线程的生命周期是由线程池管理的，也就是说即使业务代码正常执行结束，新线程能否销毁，
取决于线程池的策略，而这种新线程销毁的不确定性，造成了新线程可能被复用，无意间将上下文信息泄露出去。

退一步说，我们的分布式服务使用线程池的时候，也可能复用线程池中的线程，那么就不会发生新线程的创建，
也就不会发生隐式传递事务上下文的动作，那么新线程就获取不到事务的上下文信息。

综上，如果在分布式事务中，使用线程池，如果需要传递事务上下文，使用隐式的方法几乎都是问题，
因此如果存在这种需求，最后显示的设置上下文信息，并且执行结束时，主动清理上下文信息。

在我们的例子里，上述代码其实不需要分布式事务的，显示的将上下文清理掉，所以最小的修复如下：
```java
// 并发执行检查
forkJoinConfig.getForkJoinPool().submit(() -> {
  list.parallelStream().forEach(checker -> {
    DTXLocalContext.set(null);
    checker.check(param, result);
  });
}).join();
```

----------

    代码版本：
    druid：1.1.16
    tx-lcn：6.0.20.RELEASE

