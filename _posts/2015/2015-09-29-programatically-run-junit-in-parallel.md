---
layout: post
title: "Programmatically run JUnit in Parallel"
comments: true
---

To speed up running tests, parallel execution is commonly used strategy.
When programmatically runnning JUnit tests via `JunitCore`, the code have to incorporate with the parallel execution logic as well.

In junit 4.12, there is a class called `org.junit.experimental.ParallelComputer` to parallel run tests.

However, this class is not much usable. First, maybe this is a trivial reason though, it is an experimental feature as the package name indicates.
The biggest problem is that `ParallelComputer` does not give any configuration. It just makes tests run in parallel either by classes or methods.

<br/>

Alternatively, I found a parallel run implementation in [maven surefire](https://maven.apache.org/surefire/maven-surefire-plugin/).

-  *`org.apache.maven.surefire.junitcore.pc.ParallelComputer`*
-  *`org.apache.maven.surefire.junitcore.pc.ParallelComputerBuilder`*

They can be used in conjunction with `JUnitCore`.

<br/>

**Here is how:**


Add a dependency to one of the surefire libraries which includes parallel run implementation.

```xml
<dependency>
  <groupId>org.apache.maven.surefire</groupId>
  <artifactId>surefire-junit47</artifactId>
  <version>2.18.1</version>
</dependency>
```

In code, prepare properties, build `ParallelComputer`, then pass it to `junitCore`.

```java
// run junit with concurrency configured
Properties properties = new Properties();
properties.put(JUnitCoreParameters.PARALLEL_KEY, "classes");
properties.put(JUnitCoreParameters.THREADCOUNT_KEY, String.valueOf(numOfThread));
properties.put(JUnitCoreParameters.PERCORETHREADCOUNT_KEY, "false");  // default is true

JUnitCoreParameters parameters = new JUnitCoreParameters(properties);
ParallelComputerBuilder builder = new ParallelComputerBuilder(
  new DefaultConsoleReporter(System.out), parameters);
ParallelComputer computer = builder.buildComputer();

// run tests in parallel
Result result = jUnitCore.run(computer, MyTestSuite.class);
```

<br/>

 To figure out parameters, you may need to see actual implementation and [surefire API web page](https://maven.apache.org/surefire/maven-surefire-plugin/test-mojo.html).  
The `ParallelComputer` in surefire is widely used via `mvn test`, and is powerfull enough to make unittests run in parallel.

<br/>
