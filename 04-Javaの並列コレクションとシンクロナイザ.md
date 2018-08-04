# 04.Javaの並列コレクションとシンクロナイザ

## Javaの並行コレクション

並行処理を補助するライブラリが Java6 で実装された

- java.util.concurrent
- java.util.concurrent.atomic
- java.util.concurrent.locks

- ConcurrentHashMap    : 更新の並行性を持つ連想配列
- CopyOnWriteArrayList : すべての操作がスレッドセーフなArrayList
- BlockingQueue        : ブロッキング操作が利用できるスレッドセーフなキュー

## BlockingQueue

プロデューサー・コンシューマー型のワーカースレッドパターン

- メインスレッドは、キューに仕事を入れる
- ワーカースレッドは、キューから仕事をとって処理する

```scala
import java.util.concurrent.ArrayBlockingQueue
import java.util.concurrent.atomic.AtomicInteger

// プロデューサー・コンシューマーパターン
  // 仕事を作る人 : メインスレッド
  // 仕事をする人 : ワーカースレッド
  // データ。プロデューサーによって生産され、コンシューマーによって加工される。
  // テーブル。データの置かれるところ。

// 4つのワーカースレッドで同時並行して仕事をやらせる
object BlockingQueueStudy extends App {
  // 10の容量を持つブロッキングキュー
    // put : 10個を超える Runnable のインスタンスを入れようとした場合に、ブロックし続ける。(仕事の追加)
    // take: 要素が0の場合はデータが挿入されるまで待ち続ける。(仕事の取り出し)
  val blockingQueue = new ArrayBlockingQueue[Runnable](10)
  val finishedCount = new AtomicInteger(0)
  var threads = Seq[Thread]()

  // 4つのワーカースレッドを作成する。
  // 4つのワーカースレッドは、テーブルから、仕事を取り出して仕事を開始する
    // テーブルに仕事が一つもないときは、仕事が追加されるまで待機する
    // try-catch によって、待機モードから、仕事モードに復帰する
  for (i <- 1 to 4) {
    val t = new Thread(() => {
      try {
        while (true) {
          val runnable = blockingQueue.take()
          runnable.run()
        }
      } catch {
        // ブロッキングされている処理を、ループ内に復帰する仕組み。(要するに、いつまでも待機してないで作業に復帰しなさい、ということ)
        // ここでは、ただループを抜けるだけの実装になっている
        case _: InterruptedException => // Thread.currentThread().interrupt()
      }
    })
    t.start()
    threads = threads :+ t
  }

  // 100個の仕事を追加する
  // ただし、10個溜まった段階でブロックされる。上のコードの、takeメソッドによって仕事は取り出される。
  for (i <- 1 to 100) {
    blockingQueue.put(() => {
      Thread.sleep(1000)
      println(s"Runnable: ${i} finished")
      finishedCount.incrementAndGet()
    })
  }

  // すべての仕事が終わるまで sleep し、終了すると、稼働中の threads を停止する
  while (finishedCount.get() != 100) Thread.sleep(1000)
  threads.foreach(_.interrupt())
}
```

## Java のシンクロナイザ

スレッドの制御フローを扱うことができるコレクション

- BlokingQueue
- ラッチ
- FutureTask
- セマフォ
- バリア

### ラッチ

最終ステートに到着するまで堰き止めておくゲートの働きをする部品。

用途： ロビーにプレイヤーが100人集まったらゲームを開始したい

```scala
import java.util.concurrent.CountDownLatch

// シンクロナイザ - ラッチ
  // 最終ステートに到着するまで堰き止めておくゲートの働きをする部品

// 3つスレッドの仕事が終わったあとに、最後の4つめのスレッドの仕事を処理し始める
object LatchStudy extends App {
  val latch = new CountDownLatch(3)

  for (i <- 1 to 3) {
    new Thread(() => {
      println(s"Finished and countDown! ${i}")
      latch.countDown() // カウントする
    }).start()
  }

  new Thread(() => {
    latch.await() // 待つ
    println("All tasks finished.")
  }).start()

}
```

### FutureTask

あるスレッドの計算結果が出てから、他のスレッドが get で取得できる

```scala
import java.util.concurrent.FutureTask

// 計算結果が可能な Runnable
// あるスレッドが計算した値を、他のスレッドが get でブロックとして持ち、出力する
object FutureTask extends App {

  val futureTask = new FutureTask[Int](() => {
    Thread.sleep(1000)
    println("FutureTask finished")
    2525
  })
  new Thread(futureTask).start()

  new Thread(() => {
    val result = futureTask.get()
    println(s"result: ${result}")
  }).start()

}
```

### セマフォ

リソースに対するアクセス数を制限する

用途： 有限のリソースを複数のスレッドで最大効率を目指して利用する

```scala
import java.util.concurrent.Semaphore

// リソースに対するアクセス数を制限する
object Semaphore extends App {
  val semaphore = new Semaphore(3)

  for (i <- 1 to 100) {
    new Thread(() => {
      try {
        semaphore.acquire() // 許可(実行権利?)の取得
        Thread.sleep(300)
        println(s"Thread finished. ${i}")
      } finally {
        semaphore.release() // 許可(実行権利?)の返却
      }
    }).start()
  }
}
```

### バリア

全員が揃ったときにスタートさせる

```scala
import java.util.concurrent.CyclicBarrier

object BarrierStudy extends App {

  val barrier = new CyclicBarrier(4, () => {
    println("Barrier Action!")
  })

  for(i <- 1 to 6) {
    new Thread(() => {
      println(s"Thread started. ${i}")
      Thread.sleep(300)
      barrier.await()
      Thread.sleep(300)
      println(s"Thread finished. ${i}")
    }).start()
  }

  Thread.sleep(5000)
  System.exit(0)
}
```

## 課題

### 初級

仕事中のスレッドを、インタラプションによる割り込みで中止させる

```scala
object Interruption extends App {

  val t = new Thread(() => try {
      while (true) {
        println("Sleeping...")
        Thread.sleep(1000)
      }
    } catch {
      case _: InterruptedException =>
  })
    t.start()
    t.interrupt()
}
```

### 中級

ラッチの再現(ラッチ = 最終ステートに到着するまで堰き止めておくゲートの働きをする部品)

```scala
import java.util.concurrent.{CountDownLatch, FutureTask}

object FutureTaskForLatch extends App {

  val latch = new CountDownLatch(2)

  val futureTasks = for (i <- 1 to 3)
    yield new FutureTask[Int](() => {
      // latch.countDown()
      Thread.sleep(1000)
      println(s"FutureTask ${i} finished")
      i
    })
  futureTasks.foreach((f) => new Thread(f).start())

  new Thread(() => {
    // latch.await()
   val result = futureTasks.foreach(_.get())
    println(s"All Task finished")
  }).start()

  Thread.sleep(5000)
  System.exit(0)

}
```

### 上級

CopyOnWriteArrayList と Semaphore で 10個しか要素を入れられないブロッキングキュー を再現する

```scala
import java.util.concurrent.{CopyOnWriteArrayList, Semaphore}

object QueueWithSemaphore extends App {

  val arrayList = new CopyOnWriteArrayList[Runnable]()
  val semaphore = new Semaphore(10)

  for (i <- 1 to 100) {
    arrayList.add(() => {
      try {
        semaphore.acquire()
        Thread.sleep(1000)
        println(s"Runnable: ${i} finished")
      } finally {
        semaphore.release()
      }
    })
  }

  for (i <- 1 to 20) {
    val t = new Thread(() => {
      try {
        while (true) {
          val runnable = arrayList.remove(0)
          runnable.run()
        }
      } catch {
        case _: ArrayIndexOutOfBoundsException =>
      }
    })
    t.start()
  }

}
```