---
layout: post
title: "Generate JUnit XML report from JUnitCore"
comments: true
---

`JUnitCore` class allows you to run junit tests programmatically.
However, generating JUnit XML reports is not provided out of the box.
Those reports are usually used by CI such as Jenkins.

Fortunately, [Apache Ant](http://ant.apache.org/) has `XMLJUnitResultFormatter` class that
generates XML test reports. You can add it as a library dependency to your project regardless of the usage of Ant as a build tool.

```xml
<dependency>
  <groupId>org.apache.ant</groupId>
  <artifactId>ant-junit</artifactId>
  <version>${ant.version}</version>
  <scope>test</scope>
</dependency>
```

The one big problem of `XMLJUnitResultFormatter` is that it implements `junit.framework.TestListener` interface, but `JUnitCore` only accepts `org.junit.runner.notification.RunListener` for listeners.

Thanks to [@CloudBees](https://twitter.com/CloudBees), a company behind Jenkins, there is an implementation to convert `XMLJUnitResultFormatter` to `RunListener`.

`JUnitResultFormatterAsRunListener` class is the one:  
[https://github.com/cloudbees/junit-standalone-runner/blob/master/src/main/java/com/cloudbees/junit/runner/App.java](https://github.com/cloudbees/junit-standalone-runner/blob/master/src/main/java/com/cloudbees/junit/runner/App.java#L189-L260)


To use it:

```java
junit.addListener(new JUnitResultFormatterAsRunListener(new XMLJUnitResultFormatter()) {
    @Override
    public void testStarted(Description description) throws Exception {
        formatter.setOutput(new FileOutputStream(new File(reportDir,"TEST-"+description.getDisplayName()+".xml")));
        super.testStarted(description);
    }
});
```

Copied from [here](https://github.com/cloudbees/junit-standalone-runner/blob/master/src/main/java/com/cloudbees/junit/runner/App.java#L130-L136)


Now, programmatic call to the junit tests generates test reports which Jenkins can pick up.
