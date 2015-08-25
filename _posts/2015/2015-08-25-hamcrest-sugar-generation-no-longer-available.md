---
layout: post
title: "Hamcrest Sugar Generation no longer available"
comments: true
---

Recently, I've been writing many [Hamcrest](http://hamcrest.org/JavaHamcrest/) custom matchers for [datasource-proxy](https://github.com/ttddyy/datasource-proxy). (see my previous post - [datasource-proxy 1.4: focusing on JDBC test](http://ttddyy.github.io/datasource-proxy-1.4-focusing-on-JDBC-test/))

From my old memory, [hamcrest(java)](http://hamcrest.org/JavaHamcrest/) had a code generator to aggregate all custom matcher methods. "Sugar Generation" using `@Factory` and `org.hamcrest.generator.config.XmlConfigurator` according to [the old google code documentation](https://code.google.com/p/hamcrest/wiki/Tutorial#Sugar_generation).

<br/>

After some research, I found the generator is no longer available.
> The code generation complicates the build. But it's as easy to maintain the generated classes by hand.

_From github issue: [https://github.com/hamcrest/JavaHamcrest/issues/84](https://github.com/hamcrest/JavaHamcrest/issues/84)_

Also, [commit to delete @Factory annotation](https://github.com/hamcrest/JavaHamcrest/commit/1cd817b1f8670d9520169e61cd7479477d3d8ce1)

<br/>

So, for aggregated `Matchers` class, you manage it on your own, now.  
_I like it rather than generated class :)_

<br/>

In [v2.0.0.0](http://search.maven.org/#artifactdetails|org.hamcrest|java-hamcrest|2.0.0.0|jar), generator is not there anymore.

<br/>
Since [Hamcrest](http://hamcrest.org/JavaHamcrest/) doesn't have much documentation, I am writing down the information here for reference anybody who may be looking for the hamcrest sugar generation.

<br/>
