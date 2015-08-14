---
layout: post
title: "datasource-proxy 1.4: focusing on JDBC test"
comments: true
---

Next release of [datasource-proxy](https://github.com/ttddyy/datasource-proxy)(v1.4), I am focusing on the aspect of unit testing in JDBC operations.

A while back, [Vlad Mihalcea](https://twitter.com/vlad_mihalcea) wrote [a blog post (Hibernate Facts: How to “assert” the SQL statement count)](http://vladmihalcea.com/2014/02/01/taming-jpa-with-the-sql-statement-count-validator/) and mentioned his usage of datasource-proxy.


This blog post inspired me to write APIs for query executions in unittest.

Currently, I'm working on [Hamcrest](http://hamcrest.org/JavaHamcrest/) matchers for `assertThat` in [JUnit](http://junit.org/). Once it's done, I'm also planning to write ones for [AssertJ](http://joel-costigliola.github.io/assertj/) version of `assertThat`.



Here is sample API:

## Hamcrest matchers

```java
ProxyTestDataSource ds = new ProxyTestDataSource(actualDataSource);

// ... do some JDBC operations ...

// check count of executions
assertThat(ds, totalCount(5));
assertThat(ds, selectCount(3));
assertThat(ds, insertCount(3));
assertThat(ds, allOf(updateCount(3), deleteCount(1)));

// for each execution
// query type
assertThat(ds, executions(0, is(select())));
assertThat(ds, executions(0, delete()));
assertThat(ds, executions(0, anyOf(insert(), update())));

// execution type
assertThat(ds, executions(0, statement()));
assertThat(ds, executions(0, is(batchPrepared())));
assertThat(ds, executions(0, isCallableOrBatchCallable()));
```

*Statement Execution*

```java
StatementExecution se = ds.getFirstStatement();

// check query with StringMatcher
assertThat(se, query(is("foo")));
assertThat(se, query(startsWith("foo")));
assertThat(se, queryType(QueryType.SELECT));
```

*Batch Statement Execution*

```java
StatementBatchExecution sbe = ds.getFirstBatchStatement();

// check batch queries
assertThat(sbe, queries(0, is("foo")));   // string matcher
assertThat(sbe, queries(hasItems("foo", "bar")));  // collection matcher
```

*Prepared Statement Execution*

```java
PreparedExecution pe = ds.getLastPrepared();

assertThat(pe, query(is("FOO")));

// check parameters by index
assertThat(pe, paramsByIndex(hasEntry(10, (Object) "FOO")));
assertThat(pe, param(10, is((Object) 100)));
assertThat(pe, param(10, Integer.class, is(100)));
assertThat(pe, paramAsInteger(10, is(100)));
assertThat(pe, paramNull(10));
assertThat(pe, paramNull(10, Types.INTEGER));
assertThat(pe, allOf(paramAsInteger(10, is(100)), paramAsInteger(11, is(101))));
```

*Batch Prepared Statement Execution*

```java
PreparedBatchExecution pbe = ds.getBatchPrepareds().get(0);

assertThat(pbe, query(is("FOO")));

// check batch executions
assertThat(pbe, batchSize(10));
assertThat(pbe, batch(0, param(10, Integer.class, is(100))));
```

*Callable Statement Execution*

```java
CallableExecution ce = ds.getFirstCallable();

assertThat(ce, query(is("FOO")));

// check parameters by index & name
assertThat(ce, paramNames(hasItem("foo")));
assertThat(ce, paramsByName(hasEntry("foo", (Object) "FOO")));
assertThat(ce, param("foo", is((Object) 100)));
assertThat(ce, param("foo", Integer.class, is(100)));
assertThat(ce, paramAsInteger("foo", is(100)));
assertThat(ce, paramAsInteger(10, is(100)));
assertThat(ce, paramNull("foo"));
assertThat(ce, paramNull("foo", Types.INTEGER));
assertThat(ce, allOf(paramAsInteger(10, is(100)), paramAsInteger("foo", is(100))));

// out parameters
assertThat(ce, outParamNames(hasItem("foo")));
assertThat(ce, outParamIndexes(hasItem(10)));
assertThat(ce, outParam("foo", Types.INTEGER));
assertThat(ce, outParam("foo", JDBCType.INTEGER));
assertThat(ce, outParam(10, Types.INTEGER));
assertThat(ce, outParam(10, JDBCType.INTEGER));
assertThat(ce, allOf(outParam("foo", JDBCType.INTEGER), outParam(10, Types.INTEGER)));
```

*Batch Callable Statement Execution*

```java
CallableBatchExecution cbe = ds.getLastBatchCallable();

assertThat(cbe, query(is("FOO")));

assertThat(cbe, batch(0, paramAsInteger("foo", is(100))));
assertThat(cbe, batch(0, outParam(10, Types.INTEGER)));
assertThat(cbe, batch(0, outParam("foo", JDBCType.INTEGER)));
assertThat(cbe, batch(0, allOf(paramNames(hasItem("foo")), outParam("bar", Types.INTEGER))));
```


## AssertJ assertions

_...TBD_


Let me know if you have any suggestions.

----
