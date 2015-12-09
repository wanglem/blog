title: Async Programming with Scala Future
date: 2015-11-21 10:01:49
<!-- published: false -->
tags: 
	- async
	- scala
	- play
	- api
---
Recently I have been developing a web app, which provides a single endpoint that wraps up several other micro web services for basic CRUD operations. If considering latency is important, a lot of popular web app stack/framework, like Ruby on Rails, Flask, and LAMP (most of my experience) only support blocking IO, and in my case I will have to make all API calls in sequence which could slow down the whole app.

Therefore, Async is the rescure. With the growing hotness of Scala, it supports really good interfaces on async programming. Here I'll talk about my recent dev experience with Scala and [Play Framework](https://www.playframework.com).

<!-- more -->

# Intro on Scala Future
Think of `Future[T]` is a *container* or a *tag* that wraps a piece of long lasting computation code whose result will be available at some point. Scala recognize that tag and will execute code it marked in a separete thread which is described in `ExecutionContext`, and then act on results by using callbacks. Say if your boss said you'll have a promotion on next year.
```scala Scala Future Example
import concurrent.Future
val promotion: Future[Title] = Future { 
	// return your new title
	new CTO
}

promotion.onSuccess {
	case t => println(s"Heh, Call me {$t.toString}.")
}
```

Also, to avoid the notorious callback hell problem, Scala provided multiple ways to simplify data transformation process within Future, like map, flatmap etc. From previous example, if you want to transform the return value to Boolean that just indicates if you got your promotion or no:
```scala Transform Future
import concurrent.Future
val promotion: Future[Double] = Future { 
	// return your new title
	new CTO
}

val isPromoted: Future[Boolean] = promotion.map { x =>
	// Whaaaaat? Are we in the future???
	x match {
		case t: Title => true
		case _ => false
	}
}
```
## A little on Promise
To create a future, you can use `Future` method as well as `Promise`. Promise can complete a future by either success or failure, and access it's future by calling `Promise.future`. Back to our previous example, yoru boss makes you a promise for a promotion, and will deliver it next year:
```scala Promise Example
def giveMePromo: Future[Title] = {
	val bossPromise = Promise()
	val promotion = Future {
		wait(Duration(1, YEAR))
		bossPromise.success(new CTO)
	}
	bossPromise.future
}

giveMePromo.onComplete {
	// your callbacks
}
```

Unlike Future which is a immutable object, you might already noticed that Promise is actually a writable Future's placeholder, but with single-assignment. Don't laugh too early on your promotion:
```scala Break Promise Like A BOSS
def giveMePromo: Future[Title] = {
	val bossPromise = Promise()
	val promotion = Future {
		wait(Duration(1, YEAR))
		bossPromise.fail(Reason("too young too simple, sometimes naive"))
	}
	bossPromise.future
}

giveMePromo.onComplete {
	// your callbacks
}
```

Personally I'm still not that sure where I should use Promise vs pure Future, probably I haven't encountered a use case that requires Prommise. I'll be mainly focusing on Future based programming.

In Play, the native web service client [WSRequestHolder](https://www.playframework.com/documentation/2.3.x/ScalaWS) enables Non-blocking I/O by default on common standard HTTP behavior like GET, POST, PUT and DELETE, and returns you it's corresponding response holder called `WSResponse`, which essentially is a container of Scala's Future object.

# Future Based Parallel Request
Behind Async I/O paradigm, it is all about system scalability, so that independent computation can start in parallel.

## Non-blocking I/O
For example, in Ad Tech world, a DSP that integrates with several Ad platforms needs store basic campaign info as well as references (IDs) that can use to look up data object from those platforms. A sample json might look like:

```json App Object
{
	"id" : 1
	"name" : "My Awesome Campaign",
	"start_date" : "2015-10-10 00:00:00",
	"end_date" : "2015-10-11 00:00:00",
	"platforms": [
		{
			"name" : "Adx",
			"campaign_id" : "101",
			"budget" : 1
		},
		{
			"name" : "AN",
			"campaign_id" : "505",
			"budget" : 999
		},
		...
	]
}
```
Obviously, we don't want to keep hard copy of all platform specific data, and can gather them on runtime with parallel web requests. Say you have a method takes a list of external infos and send calls to their systems:
```scala Scaling Web Requests
import play.api.libs.json._
import play.api.libs.ws.{WSResponse, WS, WSRequestHolder}
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

def callExchanges(refs: List[Info]): Future[List[WSResponse]] = {
	val results: List[Future[WSResponse]] = refs.map{ ref =>
	  sendRequest(ref.campaignId, ref.url)
	}

	// Transform List[Future[T]] to Future[List[T]] and 
	// collect the response together!
	Future.sequence(results)
}

def sendRequest(id: Int, url: String): Future[WSResponse] = {
	// a naive GET
	WS.url(s"$url/campaign/$id").get()
}

```

## Fine Control of Parallelism
While parallel requests can reduce a lot latency and improve system scalability, sometimes we also need granular control on parallelism.

Back to the campaign example, before hitting all the exchanges to get campaigns, you want to pull in auth token for each exchange, obviously they have to be in order.

Of course you can just block your Future by doing:
```scala Blocking Future
import scala.concurrent.{Await, Future}

def callExchanges(refs: List[Info]): Future[List[WSResponse]] = {
	val results: List[Future[WSResponse]] = refs.map{ ref =>
		val futureToken: Future[Token] = TokenManager.getTokenByExchange(ref.exchange)
		val token = Await.result[Token](futureToken, Duration(30, SECONDS))
		sendRequest(ref.campaignId, ref.url, token.get())
	}

	// Transform List[Future[T]] to Future[List[T]] and 
	// collect the response together!
	Future.sequence(results)
}
```
But why not just control the sequence of your Future instead of blocking the whole app?
```scala Sequencing Future
import scala.concurrent.{Await, Future}

def callExchanges(refs: List[Info]): Future[List[WSResponse]] = {
	val results: List[Future[WSResponse]] = refs.map{ ref =>
		sendRequest(ref.campaignId, ref.url, TokenManager.getTokenByExchange(ref.exchange))
	}

	// Transform List[Future[T]] to Future[List[T]] and 
	// collect the response together!
	Future.sequence(results)
}
def sendRequest(id: Int, url: String, token: Token): Future[WSResponse] = {
	for {
		tk <- token.map(get)
		rs <- WS.url(s"$url/campaign/$id").withHeader(Seq("auth" -> tk)).get()
	} yield(rs)
}
```
Whether a Future started or not really depends on statement evaluation. Note that we start the token Future at `TokenManager.getTokenByExchange(ref.exchange)` which is outside the for comprehension, while WS request start in it, so the campaign call would not be evaluated before it gets token string.

## Top to Bottom Non-blocking
One more thing worth mentioning that if you decide go with Future based non-blocking scheme, you need design your app from top to bottom with Future, which means you will have to make all function calls asynchronous and return Futures.

