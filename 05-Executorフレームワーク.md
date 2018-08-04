# 05. Executor フレームワーク

## スレッドの生成コストと安全性

スレッドはそのままでは使えない(安全性とコストに問題がある)
  - スレッドの作成上限
  - スレッドの生成、終了コストが大きい
  - スレッドのメモリ消費量

OutOfMemoryWithThread を発生させる。(インクリメントするスレッドを無限に生成する)

```scala
import java.util.concurrent.atomic.AtomicInteger

object OutOfMemoryWithThread extends App {

  val counter = new AtomicInteger(0)

  while (true) {
    new Thread(() => {
      println(counter.incrementAndGet())
      Thread.sleep(1000)
    }).start()
  }
}
```

## スレッドプール

- スレッドプール
  - 特定の個数のスレッドを待機させておく部品。
  - ただし、シャットダウンさせなければ、メモリを消費し続ける

## Executor フレームワーク

Executor: スレッドプールに仕事を代わりにさせる部品群のこと

- ExecutorService : 特定のルールを持ったスレッドプール
  - Callable : 戻り値を受け取れるRunnable

10個のスレッドが、1000個の Callable タスクを処理する。最終的に 1000回分を合計する。

タスク： count の インクリメント。

```scala
import java.util.concurrent.{Callable, Executors}
import java.util.concurrent.atomic.AtomicInteger

object ExecutorServiceStudy extends App {
  val es = Executors.newFixedThreadPool(10)
  val counter = new AtomicInteger(0)

  val futures = for (i <- 1 to 1000) yield {
    es.submit(new Callable[Int] {
      override def call(): Int = {
        val count = counter.incrementAndGet()
        println(count)
        Thread.sleep(100)
        count
      }
    })
  }

  println("sum: " + futures.foldLeft(0)((acc, f) => acc + f.get()))
  es.shutdownNow()
}
```

## 様々な種類のスレッドプール

- newFixedThreadPool      : サイズ固定
- newCachedThreadPool     : 必要に応じてスレッドを新規作成
- newSingleThreadExecutor : 単一のワーカースレッド
- newScheduledThreadPool  : 個数指定、遅延実行、周期実行

## スレッドダンプとスレッドのステータス

スレッドダンプ： すべてのスレッドの状態を出力する

``jps : スレッドダンプの出力``

スレッドの状態

- NEW           : 起動前
- RUNNABLE      : 実行中
- BLOCKED       : ブロック中かつ解除待ち
- WAITING       : 他のスレッドの特定のアクションを無制限に待つ
- TIMED_WAITING : 他のスレッドの特定のアクションを時間制限有りで待つ
- TERMINATED    : 終了状態

## VisualVM

``jvisualvm : GUI``

## 課題

### 初級

スレッド上限10個のスレッドプールで並行処理

```scala
import java.util.concurrent.Executors

object TenThousandNamePrinter2 extends App {

  val es = Executors.newFixedThreadPool(10)

  for (i <- 1 to 10000) {
    es.submit(new Runnable() {
      override def run(): Unit = {
        Thread.sleep(1000)
        println(Thread.currentThread().getName)
      }
    })
  }
}
```