# 07. Future と Promise

## 関数型プログラミングと並行処理の融合

- 非関数型プログラミングにおける並行処理の課題
  - メモリの共有、競合、可視性、ロック、逸出、コンテキストスイッチ...

## Future

- Future
  - Asyncプログラミングにおいて、終了しているかどうかわからない処理結果を抽象化した型
  - 非同期に処理される結果が入ったOption型のようなもの
  - map,flatMap,filter,forなどの式を適用できる
  - 指定した型の値が「与えられる」もの

Futureシングルトンと、成功時の処理を実装する

```scala
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

object FutureStudy extends App {

  val s = "Hello"
  // Futureシングルトンの実装。関数を与えると、その関数を非同期に与えるFuture[+T]を返す。
  // 1秒後に文字列の連結を非同期に行う。(Future[STring]を返す)
  val f: Future[String] = Future {
    Thread.sleep(1000)
    s + " future!"
  }
  // Future成功時の処理
    // Futureは、定義部分が呼び出されたときに自動実行される。
  f.foreach { case s: String =>
    println(s)
  }

  println(f.isCompleted) // false

  // Futureの結果取得を待つ
  Thread.sleep(5000) // Hello future!

  // 待機中に、futureの非同期処理は完了している。
  println(f.isCompleted) // true

  // 同様にFutureを定義する。1秒待機した後に、例外を発生させる処理を行う。
  val f2: Future[String] = Future {
    Thread.sleep(1000)
    throw new RuntimeException("わざと失敗")
  }

  // Futureの実行。(宣言を呼び出す)。Throwableなら、例外名を出力する。
  f2.failed.foreach { case e: Throwable =>
    println(e.getMessage)
  }

  println(f2.isCompleted) // false

  // 待機中に終了する
  Thread.sleep(5000) // わざと失敗

  // 例外が発生したとしても、Futureは「完了」している
  println(f2.isCompleted) // true

}
// false
// Hello future!
// true
// false
// わざと失敗
// true
```

- 上のコードを、Awaitで書き直す。実際にどのようなスレッドでFutureが実行されているか出力される

```scala
import scala.concurrent.{Await, Future}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._
import scala.language.postfixOps

object FutureStudy extends App {

  val s = "Hello"
  val f: Future[String] = Future {
    Thread.sleep(1000)
    // ForkJoinPool-1-woroker-5 スレッドで実行される
    println(s"[ThreadName] In Future: ${Thread.currentThread.getName}")
    s + " future!"

  }

  f.foreach { case s: String =>
    // ForkJoinPool-1-woroker-5 スレッドで実行される
    println(s"[ThreadName] In Success: ${Thread.currentThread.getName}")
    println(s)
  }

  println(f.isCompleted) // false

  Await.ready(f, 5000 millisecond) // Hello future!

  // [ThreadName] In App: main
  println(s"[ThreadName] In App: ${Thread.currentThread.getName}")
  println(f.isCompleted) // true

}
// false
// [ThreadName] In Future: ForkJoinPool-1-worker-5
// [ThreadName] In App: main
// true
// [ThreadName] In Success: ForkJoinPool-1-worker-5
// Hello future!
```

ExecutionContext: Thread に Runnable(タスク)を、上限付きプールから、いい感じに振り分けてくれる。

- Future を用いると、並行処理プログラミングが実現されている
  - import scala.concurrent.ExecutionContext.Implicits.global で、暗黙的に ExecutorContext を使っている
  - 内部的には、Java の ExecutorService によって並行処理プログラミングを実現している

実際

```scala
val p: Promise[String] = Promise[String]
val a: Future[String] = p.future
a.foreach(println)
scala.concurrent.ExecutionContext.Implicits.global.execute(
  new Runnable{def run = {
    Thread.sleep(3000)
    p.success("fuga")
  }}
)
```

省略

```scala
import scala.concurrent.ExecutionContext.Implicits.global
val a: Future[String] = Future {
  Thread.sleep(3000)
  "fuga"
}
a.foreach(println)
```

## Future の Option としての性質

Future の実行結果の成否によって、処理を振り分ける。Future に適用する関数の中の Future に対しては flatMap を利用する

```scala
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.Future
import scala.util.{Failure, Random, Success}

// 予約
object FutureOptionUsage extends App {

  val random = new Random()
  val waitMaxMilliSec = 3000

  val futureMilliSec: Future[Int] = Future {
    val waitMilliSec = random.nextInt(waitMaxMilliSec)
    if (waitMilliSec < 1000) throw new RuntimeException(s"waitMilliSec is ${waitMilliSec}")
    Thread.sleep(waitMilliSec)
    waitMilliSec
  }

  val futureSec: Future[Double] = futureMilliSec.map(i => i.toDouble / 1000)

  // onComplete パターンマッチ
  futureSec onComplete {
    case Success(waitSec) => println(s"Success! ${waitSec} sec")
    case Failure(t) =>  println(s"Failure: ${t.getMessage}")
  }

  Thread.sleep(3000)
}
```

## Future の組み合わせ

for式で、複数の Future を組み合わせる

```scala
import scala.concurrent.ExecutionContext.Implicits.global

import scala.concurrent.Future
import scala.util.{Failure, Random, Success}

// 複数予約
object CompositeFuture extends App {
  val random = new Random()
  val waitMaxMilliSec = 3000

  def waitRandom(futureName: String): Int = {
    val waitMilliSec = random.nextInt(waitMaxMilliSec)
    if (waitMilliSec < 500) throw new RuntimeException(s"${futureName} waitMilliSec is ${waitMilliSec}")
    Thread.sleep(waitMilliSec)
    waitMilliSec
  }

  val futureFirst: Future[Int] = Future {
    waitRandom("first")
  }
  val futureSecond: Future[Int] = Future {
    waitRandom("second")
  }

  val compositeFuture: Future[(Int, Int)] = for {
    first <- futureFirst
    second <- futureSecond
  } yield (first, second)

  compositeFuture onComplete {
    case Success((first, second)) => println(s"Success! first:${first} second ${second}")
    case Failure(t) => println(s"Failure: ${t.getMessage}")
  }

  Thread.sleep(5000)
}
```

## Promise

- Promise
  - 成功あるいは失敗を表す値を設定することによって Future に変換することのできるクラス
  - 既存の処理などを非同期にして Future を作るときに使う
  - Future の中間を媒介できるもの?
  - 指定した型の値を「与える」もの

```scala
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.{Await, Future, Promise}
import scala.concurrent.duration._

object PromiseStudy extends App {
  val promiseGetInt: Promise[Int] = Promise[Int]
  val futureByPromise: Future[Int] = promiseGetInt.future // PromiseからFutureを作ることが出来る

  // Promiseが解決されたときの処理を、Futureで書く
  val mappedFuture = futureByPromise.map { i =>
    println(s"Success! i: ${i}")
  }

  // 別スレッドで重い処理をして、終わったらPromiseに値を返す
  Future {
    Thread.sleep(300)
    promiseGetInt.success(1)
  }

  Await.ready(mappedFuture, 5000.millisecond)
}
```

callback を指定するタイプの非同期処理をラップして Future を返す

```scala
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.{Await, Future, Promise}
import scala.concurrent.duration.Duration
import scala.util.Random

// Calback
class CallbackSomething {
  val random = new Random()

  // 成否に応じた処理
  def doSomething(onSuccess: Int => Unit, onFailure: Throwable => Unit): Unit = {
    val i = random.nextInt(10)
    // 5未満なら成功時の処理、5より大きければ失敗
    if (i < 5) onSuccess(i) else onFailure(new RuntimeException(i.toString))
  }
}

// Future
class FutureSomething {
  // callback を生成
  val callbackSomething  = new CallbackSomething

  def doSomething(): Future[Int] = {
    val promise = Promise[Int]
    callbackSomething.doSomething(i => promise.success(i), t => promise.failure(t))
    promise.future
  }
}

object CallbackFuture extends App {
  val futureSomething = new FutureSomething

  val iFuture = futureSomething.doSomething()
  val jFuture = futureSomething.doSomething()

  val iplusj = for {
    i <- iFuture
    j <- jFuture
  } yield i + j

  val result = Await.result(iplusj, Duration.Inf)
  println(result)
}
```

## 課題

### 初級

Future の結果を組み合わせる

```scala
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration.Duration
import scala.concurrent.{Await, Future}
import scala.util.{Failure, Random, Success}

object CompositeFutureChallenge extends App {

  val random = new Random()
  // 未確定の値を Future型 に格納する
  val f1 = Future {
    random.nextInt(10)
  }
  val f2 = Future {
    random.nextInt(10)
  }
  val f3 = Future {
    random.nextInt(10)
  }
  val f4 = Future {
    random.nextInt(10)
  }

  // 未確定の値を組み合わせる
  val compositeFuture: Future[Int] = for {
    first <- f1
    second <- f2
    third <- f3
    forth <- f4
  } yield first * second * third * forth

  // 確定時の成否によって、処理を振り分ける
  compositeFuture onComplete {
    case Success(value) => println(value)
    case Failure(throwable) => throwable.printStackTrace()
  }
  Await.result(compositeFuture, Duration.Inf)
}
```

### 中級

Promise を利用して、コールバック関数の結果を Future型で取得する

```scala
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration.Duration
import scala.concurrent.{Await, Future, Promise}
import scala.util.{Failure, Success}

object PromiseStdIn extends App {

  def applyFromStdIn(lineInputProcessor: Int => Unit): Unit = {
    lineInputProcessor(io.StdIn.readLine().toInt)
  }

  val promise: Promise[Int] = Promise[Int]
  applyFromStdIn((i) => promise.success(i * 7))
  val future: Future[Int] = promise.future

  future onComplete {
    case Success(value) => println(value)
    case Failure(throwable) => throwable.printStackTrace()
  }
  Await.result(future, Duration.Inf)
}
```

### 上級
