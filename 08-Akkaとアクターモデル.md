# Akka とアクターモデル

## Akka とは

Akka: 分散実行によるスケーラビリティと耐障害性の性質を持ったツールキット

## アクターモデルとは

- Eralng : Open Telecom Platform
- AkkaActor

1. Message
2. MailBox
3. Actor

BlokingQueue と ExecutorService を利用した並行処理との違い

- メモリの同期化や可視性を考えなくてよい
  - メッセージの送信: 非同期
  - メッセージの処理: 同期的にループで逐次処理

### アクターモデルの分散実行の仕組み

- メッセージ以外で外部との依存関係を持たない
  - 全てのアクターは、強くカプセル化され、パス(メールアドレス)を持ち、互いにメッセージを送り合う

- ActorSystem
  - Actor
    - childActor
    - grandChild

### アクターモデルの耐障害性の仕組み

- Supervision hierarchies: アクターの階層
  - Supervisor: アクターの監視オブジェクト
    - Guradian Actor : ユーザーの作成したアクターの親につく Supervisor

Let it Clash : Supervisor は アクターが不審な挙動をすれば、接続ごと破棄して、新たに正しく接続し直して仕事を再開させる

## Akka Actor のセットアップ

アクターにメッセージを送り、アクターがメッセージを処理する。

ActorRef: アクターの参照

```scala
import akka.actor.{Actor, ActorSystem, Props}
import akka.event.Logging

import scala.concurrent.Await
import scala.concurrent.duration._

// メールボックスからメッセージを受け取る Actor
class MyActor extends Actor {
  val log = Logging(context.system, this)

  // receive メソッドでメッセージを処理する
  def receive = {
    case "test" => log.info("received test")
    case _ => log.info("received unknown message")
  }
}

object ActorStudy extends App {
  // ActorSystem の生成
  val system = ActorSystem("actorStudy")

  // Actor の生成
  // Proops: Actor生成時のオプション。Actorのクラス、引数となるオブジェクトを指定したりする。Actorを作る際の使いまわせるレシピ
  // actorOfメソッド: Actorを作成し、 ActorRef というアクターの参照を取得する
  val myActor = system.actorOf(Props[MyActor], "myActor")
  // myActor に対して、"test" と "hoge" というメッセージを送る
  // ! は tell メソッド = myActor.tell("test",Actor.noSender)
    // Akka以外からメッセージを送る場合
      // 実行結果を Future で受け取れるように ask あるいは Inbox を用いる。ただし原則は Tell don't Ask
  myActor ! "test"
  myActor ! "hoge"

  // terminate : すべての ActorSystem を終了する
  Await.ready(system.terminate(), 5000.milliseconds)
}
```

## Actor からの返信を受け取る

Inbox を利用して、アクター以外の処理からメッセージを送る。アクターからの返信を受け取り、出力する

```scala
import akka.actor.{Actor, ActorSystem, Inbox, Props}
import scala.concurrent.duration._

// メッセージの定義
case object Greet
case class WhoToGreet(who: String)
case class Greeting(message: String)

// 挨拶を行うアクターの定義
class Greeter extends Actor {
  var greeting = ""

  def receive = {
    case WhoToGreet(who) => greeting = s"hello, $who"
    case Greet => sender ! Greeting(greeting)
  }
}

class GreetPrinter extends Actor {
  def receive = {
    case Greeting(message) => println(message)
  }
}

object HelloAkkaScala extends App {
  val system = ActorSystem("helloAkka")

  val greeter = system.actorOf(Props[Greeter], "greeter")

  val inbox = Inbox.create(system)

  greeter ! WhoToGreet("akka")
  // Inbox から Greeter アクターがアクター内で参照できるようにした上で、メッセージを送る
  inbox.send(greeter, Greet)
  // 5秒間メッセージの受取があるかを待つ。GreeterアクターのGreetingメッセージを待つ。
  val Greeting(message1) = inbox.receive(5.seconds)
  println(s"Greeting: $message1")

  greeter ! WhoToGreet("Lightbend")
  inbox.send(greeter, Greet)
  val Greeting(message2) = inbox.receive(5.seconds)
  println(s"Greeting: $message2")

  val greetPrinter = system.actorOf(Props[GreetPrinter])
  // スケジューラーを利用したメッセージの送信
  // 左から
    // 送信時の最初のディレイ
    // 送信のインターバル
    // 受け取り手の ActorRef。(参照) : 住所
    // メッセージのオブジェクト : 手紙
    // 実行コンテキスト
    // 送り手のアクター
  // dispatcher : 処理を待っているデータやプロセスが複数ある場合に、効率よく処理できるよう必要な資源の振り分けや割り当てを行うシステムやプログラムのこと
  system.scheduler.schedule(0.seconds, 1.second, greeter, Greet)(system.dispatcher, greetPrinter)
}
```

## Akka Actor でできること

メッセージの信頼性

- at-most-onece: 0 or 1
- at-least-onece: 最低 1
- exactly-once: 確実に 1

基本的にメッセージは消失する可能性がある、という考えの元に構築される

## 課題

### 初級

メッセージ受信回数を出力するアクターに、1万回メッセージを送信する

```scala
import akka.actor.{Actor, ActorSystem, Props}

import scala.concurrent.Await
import scala.concurrent.duration._

class MessageCountActor extends Actor {
  var count = 0

  def receive = {
    case _ => {
      count = count + 1
      println(count)
    }
  }
}

object MessageCountActorApp extends App {
  val system = ActorSystem("messageCountActorApp")

  val myActor = system.actorOf(Props[MessageCountActor], "messageCountActor")

  for (i <- 1 to 10000) {
    myActor ! "test"
  }

  Await.ready(system.terminate(), Duration.Inf)
}
```

### 中級

互いに返信し合うアクターを実装する。

tellメソッドで、アクターから別のアクターにメッセージを送信できる

```scala
import akka.actor.{Actor, ActorSystem, Props}
import akka.event.Logging

import scala.concurrent.Await
import scala.concurrent.duration._

case object Ping
case object Pong

class Actor_A extends Actor {
  var log = Logging(context.system, this)

  def receive = {
    case Ping => {
      log.info("ping")
      sender() ! Pong
    }
    case _ =>
  }
}

class Actor_B extends Actor {
  var log = Logging(context.system, this)

  def receive = {
    case Pong => {
      log.info("pong")
      sender() ! Ping
    }
    case _ =>
  }
}

object ReplayActorApp extends App {
  val system = ActorSystem("replayActorApp")

  val myActor_A = system.actorOf(Props[Actor_A], "myActor_A")
  val myActor_B = system.actorOf(Props[Actor_B], "myActor_B")

  myActor_A.tell(Ping,myActor_B)

  Await.ready(system.terminate(), 15.seconds)
}
```

### 上級

重い処理を担うアクターに、大量のメッセージを送る。結果、メッセージの受信と処理に遅延が生じる。

```scala
import java.util.Date

import akka.actor.{Actor, ActorSystem, Props}
import akka.event.Logging


class HeavyActor extends Actor {
  var log = Logging(context.system, this)

  def receive = {
    case message:String =>
      log.info(message)
      Thread.sleep(1000)
  }
}

object TooMuchMessageApp extends App {
  val system = ActorSystem("TooMuchMessageApp")

  val myActor = system.actorOf(Props[HeavyActor], "myActor")

  while (true) {
    myActor ! new Date().toString
  }
}
```
