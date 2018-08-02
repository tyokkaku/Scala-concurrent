# 09. AkkaActor の耐障害性

## スーパーバイザーの取れるアクション

## アクターのライフサイクル

## 障害のハンドリング

Supervisor

```scala
import akka.actor.Actor
import scala.concurrent.duration._
import akka.actor.{OneForOneStrategy, Props}
import akka.actor.SupervisorStrategy.{Escalate, Restart, Resume, Stop}

// 指定された Props をメッセージとして受け取り、送り手に作成した子アクターの参照を送り返す
class Supervisor extends Actor {

  override val supervisorStrategy =
    // 1分間 に 10回 の再起動を許容する
    OneForOneStrategy(maxNrOfRetries = 10, withinTimeRange = 1 minute) {
      case _: ArithmeticException => Resume // 続行命令。エラーを無視して次のメッセージを続けて処理する
      case _: NullPointerException => Restart // 再起動命令。アクターを作り直してメッセージの処理を再開する
      case _: IllegalArgumentException => Stop // 停止命令。アクターの活動を停止する
      case _: Exception => Escalate // 委譲命令。親のアクターにエラーを受け渡して対処させる
    }

  def receive = {
    // 自身の context を利用して、子アクターを作成する
    case p: Props => sender() ! context.actorOf(p)
  }
}
```

childActor

```scala
import akka.actor.Actor

class Child extends Actor {
  var state = 0
  def receive = {
    case ex: Exception => throw ex // 例外発生じは例外を投げるのみ
    case x: Int => state = x
    case "get" => sender() ! state
  }
}
```

App

```scala
import akka.actor.{ActorRef, ActorSystem, Inbox, Props}

import scala.concurrent.duration._
import scala.concurrent.Await
import scala.concurrent.duration.Duration

object FaultHandlingStudy extends App {

  val system = ActorSystem("faultHandlingStudy")
  val inbox = Inbox.create(system)
  implicit val sender = inbox.getRef()

  val supervisor = system.actorOf(Props[Supervisor], "supervisor")

  supervisor ! Props[Child]
  val child = inbox.receive(5.seconds).asInstanceOf[ActorRef]

  // 通常の動作確認。子アクターに 42 をセットする
  child ! 42 // set state to 42
  child ! "get"
  println("set state to 42: " + inbox.receive(5.seconds)) // 42 expected

  // 続行命令の確認。子アクターで例外を発生させるても、続行し、状態は変わっていない。
  child ! new ArithmeticException // crash it
  child ! "get"
  println("crash it: " + inbox.receive(5.seconds)) // 42 expected

  // 再起動命令の確認。
  child ! new NullPointerException // crash it harder
  child ! "get"
  println("crash it harder: " + inbox.receive(5.seconds)) // 0 expected

  // 停止命令 と watch の動作確認
    // watch: Actorを監視する。Actorが終了した場合に Terminated というオブジェクトをメッセージとして受け取ることができる。
  // watch and break it: Terminated(Actor[akka://faultHandlingStudy/user/supervisor/$a#702501537])
  inbox.watch(child) // have Inbox watch “child”
  child ! new IllegalArgumentException // break it
  println("watch and break it: " + inbox.receive(5.seconds))

  // Actor の新規作成
  supervisor ! Props[Child]
  val child2 = inbox.receive(5.seconds).asInstanceOf[ActorRef]
  inbox.watch(child2)
  child2 ! "get" // verify it is alive
  println("new child: " + inbox.receive(5.seconds)) // 0 expected

  // 委譲命令の確認。Exeption を supervisor に委譲する。supervisor は、エラーログを出力して再起動する。
  child2 ! new Exception("CRASH") // escalate failure
  println("escalate failure: " + inbox.receive(5.seconds))
  /*
    supervisor でエラーを処理したことがわかる
    [ERROR] [08/02/2018 17:31:18.444] [faultHandlingStudy-akka.actor.default-dispatcher-4]    [akka://faultHandlingStudy/user/supervisor] CRASH
    java.lang.Exception: CRASH
    	at FaultHandlingStudy$.delayedEndpoint$FaultHandlingStudy$1(FaultHandlingStudy.scala:40)
    	at FaultHandlingStudy$delayedInit$body.apply(FaultHandlingStudy.scala:7)
    	at scala.Function0.apply$mcV$sp(Function0.scala:34)
    	at scala.Function0.apply$mcV$sp$(Function0.scala:34)
    	at scala.runtime.AbstractFunction0.apply$mcV$sp(AbstractFunction0.scala:12)
    	at scala.App.$anonfun$main$1$adapted(App.scala:76)
    	at scala.collection.immutable.List.foreach(List.scala:389)
    	at scala.App.main(App.scala:76)
    	at scala.App.main$(App.scala:74)
    	at FaultHandlingStudy$.main(FaultHandlingStudy.scala:7)
    	at FaultHandlingStudy.main(FaultHandlingStudy.scala)
  */

  Await.ready(system.terminate(), Duration.Inf)
}
```

## アクターの止め方

## 子アクターを利用した分散処理

```scala
import akka.actor.SupervisorStrategy.{Restart, Stop}
import akka.actor.{Actor, ActorInitializationException, ActorKilledException, ActorSystem, DeathPactException, Inbox, OneForOneStrategy, Props}
import akka.routing.{ActorRefRoutee, RoundRobinRoutingLogic, Router}

import scala.concurrent.Await
import scala.concurrent.duration._
import scala.util.{Failure, Random, Success, Try}

case class DivideRandomMessage(numerator: Int)  // 親から子に割り算を指示
case class AnswerMessage(num: Int)  // 答えの取得
case class ListDivideRandomMessage(numeratorList: Seq[Int]) // 親への指示

// RandomDivider: ランダムな割る数 (0 ~ 3) で 与えられる分子の計算処理をする

class RandomDivider extends Actor {
  val random = new Random()
  val denominator = random.nextInt(3) // 0, 1, 2のどれかで割る。 0 で割るアクターは壊れている

  def receive = {// 割る
    case m@DivideRandomMessage(numerator) =>
      val answer = Try {
        AnswerMessage(numerator / denominator)
      } match {
        case Success(a) => a
        case Failure(e) =>
          // もし例外が発生したら、再起動後の次回分のために、今回分の数値を送信する
            // forward: 送り主の値を変更することなく転送する
          self.forward(m)
          throw e
      }
      println(s" ${numerator} / ${denominator} is ${answer}")
      sender() ! answer
  }

}

// ListRandomDivider: 親アクター。子にリストを与え計算させ、その合計を求める

class ListRandomDivider extends Actor {
  // Listメッセージの送信者の参照を取得する
  var listDivideMessageSender = Actor.noSender
  var sum = 0
  var answerCount = 0
  var totalAnswerCount = 0

  override val supervisorStrategy =
    // 10秒間に最大10回のリトライする
    OneForOneStrategy(maxNrOfRetries = 10, 10.seconds) {
      case _: ArithmeticException => {
        println("Restart by ArithmeticException")
        Restart
      }
      case _: ActorInitializationException => Stop
      case _: ActorKilledException => Stop
      case _: DeathPactException => Stop
      case _: Exception => Restart
    }

  // 4つのアクターを生成し、1つ1つを巡回しながらメッセージを送る
    // Router: 複数の子アクターにメッセージを送る方法を定義する
    // RoundRobinRoutingLogic: 1つ1つ巡回
  val router = {
    val routees = Vector.fill(4) {
      ActorRefRoutee(context.actorOf(Props[RandomDivider]))
    }
    Router(RoundRobinRoutingLogic(), routees)
  }

  def receive = {
    // リストを受信したら、4つのアクターに、リストの要素を1つずつ送信する
    case ListDivideRandomMessage(numeratorList) => {
      listDivideMessageSender = sender()
      totalAnswerCount = numeratorList.size
      numeratorList.foreach(n => router.route(DivideRandomMessage(n), self))
    }
    // 答えを受信したら、合計を計算する
    case AnswerMessage(num) => {
      sum += num
      answerCount += 1
      if (answerCount == totalAnswerCount) listDivideMessageSender ! sum
    }
  }
}

object RandomDivide extends App {
  val system = ActorSystem("randomDivide")
  val inbox = Inbox.create(system)
  implicit val sender = inbox.getRef()

  val listRandomDivider = system.actorOf(Props[ListRandomDivider], "listRandomDivider")
  listRandomDivider ! ListDivideRandomMessage(Seq(1, 2, 3, 4))
  val result = inbox.receive(10.seconds)
  println(s"Result: ${result}")

  Await.ready(system.terminate(), Duration.Inf)
}
```

## アクターの状態の変更