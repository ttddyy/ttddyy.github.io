---
layout: post
title: "Executable Jar for Integration Test"
comments: true
---

Recently, I wrote an integration test infrastructure at work.  
Since integration test is different from simple unittests, we made our integration tests to be an executable jar.  
In this blog post, I'd like to write about why we made executable jar and some of the considerations to achieve it.


## Integration Test
First, the word `"integration test"` is very vague.

To clarify, in my case, integration test was:

- **run group of tests against externally running application**

So, it is a black box test.


# Integration tests as an executable jar

Instead of running tests from maven command such like `mvn test` or `mvn verify`, we generate an executable jar from integration test project.  
There are multiple reasons, but first I want to describe our build infrastructure.

We use [Jenkins][jenkins] and [Artifactory][artifactory] for our build.


<br/>

Our Jenkins jobs do followings:

1. compile and *run unittests*
1. deploy the artifacts to Artifactory as a build
1. provision a new VM
1. run the latest app resolved from Artifactory by specifying the build id
1. *run integration tests* against the latest app

<br/>


The benefits of using Artifactory for binary management are:

- Don't need to checkout entire project to run integration test
- Don't need to compile project to run integration test
- Easy to provision VM with specific version of app using the build ID
- Accurately reproduce environment(application version and integration test version) by retrieving binaries by build ID

By making integration test as an executable jar, we can retrieve binary from Artifactory and simply execute it in Jenkins Job.


# To run tests

We use JUnit4 to run integration tests. In the main method of the executable jar, it simply calls `JUnitCore` to programmatically invoke all tests.  

However, just invoking tests is not enough for integration test.  
We have to consider followings:

- generate test reports
- code coverage
- parallel test run

## Generate test reports

To show up the test results in Jenkins, running integration tests from executable jar needs to produce XML reports.

If `maven`, `gradle`, `ants` are used to run tests, they by default or configuration generate XML reports.
However, since we run tests from executable jar, we programmatically run tests using `JUnitCore`.
Unfortunately, junit4 doesn't come with XML reporting feature for Jenkins to pick up.

I found `XMLJUnitResultFormatter` class in [Apache Ant][ant], however the class cannot directly used by `JUnitCore` due to the difference of interfaces.  
I wrote a blog post ["Generate JUnit XML report from JUnitCore"](/generate-junit-xml-report-from-junitcore/).
There, you can find how to integrate `XMLJUnitResultFormatter` to `JUnitCore` in detail.

Once executable jar starts producing XML reports, Jenkins can accurately show the result report same as unittests triggered by maven/gradle/ant.

## Code coverage report

We use [JaCoCo](http://eclemma.org/jacoco/) to generate code coverage for tests.  
Our web application is also an executable jar.
So, simply attach JaCoCo agent to JVM when we start app for integration test.  
The generated report files can be seen in Jenkins using [JaCoCo Jenkins Plugin](https://wiki.jenkins-ci.org/display/JENKINS/JaCoCo+Plugin).

```bash
> java -javaagent:jacocoagent.jar=destfile=integration-test.exec our-application.jar
```


## Parallel test run

Usually integration test is a long running test. Thus, parallel run is necessary to shorten total run time.

Still POC but I was able to parallel run the tests by providing N number of environments, as well as configuring `JUnitCore` to concurrently run tests.  
I have summarized the setup in my blog post: ["Programmatically run JUnit in Parallel"](/programatically-run-junit-in-parallel/)


# Summary
So far, we are happy with current integration test setup.

I had to deep dive into JUnit4 a little bit for test report generation and parallel run.
Hopefully in [JUnit5 (JUnit-Lambda)][junit-lambda], setup becomes straightforward, configurable, and easily achievable.

<br/>


[artifactory]: https://www.jfrog.com/artifactory/
[jenkins]: https://jenkins-ci.org/
[ant]: http://ant.apache.org/
[junit-lambda]: http://junit.org/junit-lambda.html
