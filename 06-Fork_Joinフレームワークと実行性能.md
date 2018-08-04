# 06. Fork/Join フレームワークと実行性能

## Fork/Join フレームワーク

Executor フレームワーク : スレッドプールを用いたスレッドの上限制限を設ける技術

Fork/Join フレームワーク: 正しくタスクを分割する技術

work-stealing アルゴリズム : 各スレッドに用意されたタスクキューで、いずけかのスレッドのタスクがキューから枯渇した時、他のスレッドからタスクを奪い取る

## Fork/Join フレームワークの使い方

タスクの実行

- execute : 非同期実行
- invoke  : 同期実行
- submit  : 実行し、Futureのインスタンスを取得

タスクの戻り値

- RecursiveTask   : 結果を返す
- RecursiveAction : 結果を返さない

並行処理

- fork : ForkJoinPool を利用した日どき実行
- join : 同期的に結果の取得


## 階乗の数の集計

シングルスレッドの階乗の合計

```scala
import scala.annotation.tailrec

object FactorialSumTrial extends App {

  val length = 5000
  val list = (for (i <- 1 to length) yield BigInt(i)).toList

  @tailrec
  private[this] def factorial(i: BigInt, acc: BigInt): BigInt = if (i == 0) acc else factorial(i - 1, i * acc)

  val start = System.currentTimeMillis()
  val factorialSum = list.foldLeft(BigInt(0))((acc, n) => acc + factorial(n, 1))
  val time = System.currentTimeMillis() - start

  println(factorialSum)
  println(s"time ${time} msec")
}
```

Fork/Join フレームワークを用いたマルチスレッドで、階乗の合計

各スレッドが、タスクを分割できるまで分割していく。

```scala
import java.util.concurrent.{ForkJoinPool, RecursiveTask}

import scala.annotation.tailrec

object ForkJoinFactorialSumStudy extends App {

  val length = 5000
  val list = (for(i <- 1 to length) yield BigInt(i)).toList
  val pool = new ForkJoinPool()

  class AggregateTask(list: List[BigInt]) extends RecursiveTask[BigInt] {
    override def compute(): BigInt = {
      val n = list.length / 2
      // タスクを分割できない場合
      if (n == 0) {
        list match {
          case List() => 0
          case List(n) => factorial(n, BigInt(1))
        }
      // タスクを分割できる場合
      } else {
        // タスクの分解
        val (left, right) = list.splitAt(n)
        val leftTask = new AggregateTask(left)
        val rightTask = new AggregateTask(right)
        leftTask.fork()
        rightTask.fork()
        leftTask.join() + rightTask.join()
      }
    }

    @tailrec
    private[this] def factorial(i: BigInt, acc: BigInt): BigInt = if (i == 0) acc else factorial(i - 1, i * acc)

  }

  val start = System.currentTimeMillis()
  val factorialSum = pool.invoke(new AggregateTask(list))
  val time = System.currentTimeMillis() - start

  println(factorialSum)
  println(s"time ${time} msec")


}
```

## アムダールの法則

並列化による効率化の限度の計算

## コンテキストスイッチとメモリの同期化コスト

- 並列化のコスト
  - コンテキストスイッチ  : スレッドの切り替え。現在実行されている実行コンテキストを一度保存し、別のスレッドのために復元しなければならない
  - メモリの同期化 : 変数情報のロックと、それをスレッドが確認できるようにするにも、コストがかかる

無闇にスレッドを作成すると、かえって効率が落ちる

シングルスレッド

```scala
object SumTrial extends App {

  val length = 10000000
  val list = (for (i <- 1 to length) yield i.toLong).toList

  val start = System.currentTimeMillis()
  val sum = list.sum
  val time = System.currentTimeMillis() - start

  println(sum)
  println(s"time: ${time} msec")
}
```

マルチスレッド

```scala
import java.util.concurrent.{ForkJoinPool, RecursiveTask}

object ForkJoinSumStudy extends App {

  val length = 1000000
  val list = (for(i <- 1 to length) yield i.toLong).toList

  val pool = new ForkJoinPool()

  class AggregateTask(list: List[Long]) extends RecursiveTask[Long] {

    override def compute(): Long = {
      val n = list.length / 2
      if (n == 0) {
        list match {
          case List() => 0
          case List(n) => n
        }
      } else {
        val (left, right) = list.splitAt(n)
        val leftTask = new AggregateTask(left)
        val rightTask = new AggregateTask(right)
        leftTask.fork()
        rightTask.fork()
        leftTask.join() + rightTask.join()
      }
    }
  }

  val start = System.currentTimeMillis()
  val sum = pool.invoke(new AggregateTask(list))
  val time = System.currentTimeMillis() - start

  println(sum)
  println(s"time: ${time} msec")
}
```