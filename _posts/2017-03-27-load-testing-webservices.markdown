---
layout: post
title:  "Load Testing Webservices"
date:   2017-03-27 10:41:30 +0200
categories: gatling scala
author: Wouter van Teijlingen
---

At [Portavita][portavita] we build webservices. We need to know how our services perform and behave under heavy load. In this article I'll try to explain to you how we use Gatling to load test our webservices. [Gatling][gatling] is a load and performance testing tool that provides you with a domain specific language to write load test scenarios.

#### Introduction and basic setup
We've divided the webservices over multiple teams and each team has the responsibility to make sure the services perform as advertised. Gatling is integrated into our Jenkins setup and each night we run load tests automatically. We are automatically notified when the performance is below a certain threshold. Some projects incorporate the Gatling load test scenarios within the project, while others have separate load test projects.

One of the more interesting questions is how to determine the lower and upper bound of the performance you require. A purely analytical approach is challenging and fun, but time-consuming and probably too complex. Lots of frameworks have many dependencies and modelling all relevant software and hardware parameters is less than practical. We try to combine the analytical and the empirical approach to come up with realistic bounds.

One of the great things about Gatling is that it gives you the opportunity to test your application with many concurrent users. In one of our webservices we use the Play framework and we interface with a database. Thanks to Gatling we actually discovered a concurrency problem in the automatic management of database sessions. This particular problem was only observed if many users connected at once to the service.

#### Writing custom validators
Gatling provides you with tools to validate the responses you receive from your service. Chances are the most checks you need are already available in Gatling. In this section I'll briefly explain how you can write your own validators if you need one.

You'll have to build a class that extends Validator and that implements the apply function. In the apply function you can inspect the actual value and access your session. See below for an example implementation. 

{% highlight scala %}
class MyCustomExistsValidator[A](val session: Session) extends Validator[A] {
  val name = "MyCustomExistsValidator"

  def apply(actual: Option[A]): Validation[Option[A]] = actual match {
    case Some(value) => {
      println("value: " + value + " expected data: " + session("expected_data").as[String])
      actual.success
    }
    case None => Validator.FoundNothingFailure
  }
}
{% endhighlight %}

I suggest to keep your validators as simple as possible. If they are memory and/or compute intensive you might negatively impact your tests. Now you can easily integrate your own validators in the check of your scenario. For example:

{% highlight scala %}
...
http("Demo")
 .get("/v1/user/${with_id}")
 .check(
   bodyString.validate(new MyCustomExistsValidator[String](_))
 )
...
{% endhighlight %}

You can read more about validators on the website of [Gatling][gatling].

#### Conclusion
In this article I've briefly discussed how we use [Gatling][gatling] at [Portavita][portavita]. We're pretty happy with our decision to use Gatling for load testing. It's straightforward to use and provides us with details about the performance of the services we develop.

[gatling]: http://www.gatling.io
[portavita]: https://www.portavita.com/

