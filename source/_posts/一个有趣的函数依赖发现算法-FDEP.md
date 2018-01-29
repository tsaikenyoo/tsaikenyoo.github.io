---
title: '一个有趣的函数依赖发现算法: FDEP'
date: 2018-01-07 19:55:53
tags:
- 算法
categories: 算法
---

前段时间，和同学报名参加了云计算大赛，命题赛里第二题要求实现一个 **可并行化** 的函数依赖发现算法，我在学长的推荐下阅读了两篇论文，并使用 Scala + Spark 实现了一个函数依赖发现算法。个人觉得这个算法的思想十分有趣，故写下此篇博客，作为记录。

## 函数依赖
函数依赖的定义不在此给出，可参考赛题介绍 [链接](https://cloud.seu.edu.cn/contest/assets/img/f4/JN2.pdf)。

## 综述论文
关于函数依赖发现算法，有这样一篇综述性论文 *Functional Dependency Discovery:An Experimental Evaluation of Seven Algorithms*，论文中介绍并对比了 7 种函数依赖发现算法，我选择了其中的 FDEP 算法进行实现。

## FDEP
FDEP 的论文可参见 *Database dependency discovery: a machine learning approach*。

许多传统的函数依赖算法是基于 **格(lattice)** 的，而 FDEP 则提出了一种截然不同的思想。

在本次函数依赖发现任务中，我们所要寻发现的是最小非平凡函数依赖，而函数依赖是具有全局性的，即局部成立的函数依赖，在全局情况下未必成立。这是发现函数依赖最棘手的问题之一。

FDEP 的思想是，不直接构造函数依赖，而是通过寻找到所有的 **非函数依赖**，再由这些 **非函数依赖** 求出 **函数依赖**。而非函数依赖的特性是局部性，任意两条记录都可以生成一条或多条非函数依赖，这点恰好与函数依赖相反。这种特性也使得该算法具有一定程度的并行化能力。

### 算法流程

FDEP 算法总共分为两步：

- 构造 Negative Cover（即通过每两条记录的对比，所产生的所有最大非函数依赖）
-  构造 Positive Cover（即通过扫描 Negative Cover 里的每一条非函数依赖，来生成最小函数依赖）

下面以一个例子来说明 FDEP 算法的流程（该例为引用 FDEP 论文中的例子）

```
A B C D
0 0 0 0
1 1 0 0
0 2 0 2
1 2 3 4
```

#### Negative Cover
该例中，共有 C4^2 = 6 对记录（元组）。每对元组可以产生一个或多个非函数依赖，产生过程如下：对于每对元组，取其具有相同属性值的属性集合作为非函数依赖的左部，每个具有不同属性值的属性作为右部。

例如，第 1 和 第 2 个元组可以产生 `CD->A`、`CD->B` 这两个非函数依赖。

最终产生的所有非函数依赖中，需要消除冗余项，即保留所有 **最大非函数依赖**。

最终该例所产生的 Negative Cover 为：
`B->A`、`CD->A`、`AC->B`、`CD->B`、`A->C`、`B->C`、`AC->D`、`B->D`

#### Positive Cover
Positive Cover 的生成只需要对 Negative Cover 顺序扫描一遍即可，且与 Negative Cover 内部的排序无关。

生成过程中，对于 Negative Cover 中右部相同的非函数依赖可以划分为一组，例如 `B->A`、`CD->A` 可以划分为一组，用于产生右部为 `A` 的函数依赖。

生成 Positive Cover 的过程为（以右部为 A 为例）： 在生成过程开始之前，初始的函数依赖为 `空集 -> A`。对于每一条右部为 A 的非函数依赖，使用这条非函数依赖所提供的信息，去细化函数依赖，一直迭代，直到所有右部为 A 的非函数依赖遍历完成。

该算法的思路是这样的，Negative Cover 所包含的信息是：什么样的依赖不是函数依赖。我们知道了哪些不是函数依赖，那么也就可以说，我们知道了哪些是函数依赖（剩下的都是）。初始状态的函数依赖 `空集 -> A` 是粒度最粗的，我们希望尽可能得到细粒度的函数依赖，即精度越高越好，而每一条 非函数依赖 都为我们提供了更多细节去进行函数依赖的细化 (Specialisation)。

以右部为 A 为例来计算函数依赖，初始化的函数依赖为 `空集->A`。使用 Negtative Cover 里的 `B->A` 可以将 `空集->A` 细化为 `C->A`、`D->A`，接着读取下一条 `CD->A` 可以将 `C->A`、`D->A` 进一步细化为 `BC->A`、`BD->A`，这就是右部为 A 的函数依赖。

可见，Positive 的构造过程是一个不断细化的迭代过程，每条非函数依赖遍历一遍即可生成函数依赖。

## 代码实现
FDEP 的作者使用了C语言实现了 FDEP 算法，并使用树作为 Positive Cover 和 Negative Cover 的数据结构，代码地址可见 FDEP 论文里的链接。

我使用了 Scala + Spark 实现了并行化的 FDEP。遗憾的是，FDEP 的瓶颈在于 tuple pair（元组对）的生成，对于行数多而属性值少的情形，生成这些 pair 会消耗大量的内存。

因此，FDEP 不适合行数特别多的数据，而对于属性多而行数少的数据则十分合适。

附代码如下：

```scala
import org.apache.spark.{SparkConf, SparkContext}
import util.control.Breaks._
import scala.collection.mutable
import scala.collection.mutable.ArrayBuffer

object Main{
	def main(args: Array[String]): Unit = {
		val conf = new SparkConf().setAppName("FunctionDependency")
		val sc = new SparkContext(conf)

		val dataPath = args(0)
		val resultPath = args(1)
		val tmpPath = args(2)
		val dataset = sc.textFile(dataPath,1)
			.map(line => line.split(","))

		val attr_num: Int = dataset.take(1)(0).length
		/* R Set contains all attributes' index */
		val R: mutable.Set[Int] = mutable.Set(1 to attr_num: _*)

		/* cartesian */
		val cart = dataset.cartesian(dataset)
		println (s"${dataset.count()} => ${cart.count()}")
		val cart_test = sc.parallelize(cart.collect(),1)

		val neg_cover_dis = cart_test.map { pair =>
			val tuple_1 = pair._1
			val tuple_2 = pair._2

			var X: mutable.Set[Int] = mutable.Set()
			var A_buffer: mutable.Set[Int] = mutable.Set()

			// initialize X and A_buffer
			for (i <- tuple_1.indices) {
				if (tuple_1(i) == tuple_2(i)) {
					X += i+1
				}
				else {
					A_buffer += i+1
				}
			}

			(X, A_buffer)

		}.reduceByKey(_ ++ _)
			.flatMap { line =>
				line._2.map(a => (a, ArrayBuffer[mutable.Set[Int]](line._1)))
			}.reduceByKey(_ ++ _)
	    	.map { line =>   // save the maximum neg_cover
				val new_set_buffer = line._2.clone()
				for(i <- line._2.indices){
					for(j <- i+1 until line._2.size){
						if(line._2(i) subsetOf line._2(j)){
							new_set_buffer -= line._2(i)
						}
						else if(line._2(j) subsetOf line._2(i)){
							new_set_buffer -= line._2(j)
						}
					}
				}
				(line._1, new_set_buffer)
			}

		sc.broadcast(cart.collect())

		val pos_cover_dis = sc.parallelize(neg_cover_dis.collect(),1) .map{ line =>
			val key: Int = line._1
			val arr: ArrayBuffer[mutable.Set[Int]] = line._2

			var new_pos_sets = new ArrayBuffer[mutable.Set[Int]]()
			new_pos_sets += mutable.Set()

			/* for each prefix set of suffix 'key' */
			arr.foreach(neg_set => {

				// new_pos_sets is equal to tmp
				val tmp = new_pos_sets.clone()

				new_pos_sets.foreach(pos_set => {
					/* if pos_set entails neg_set */
					if(pos_set subsetOf neg_set) {
						tmp -= pos_set
						val rest_set = R -- neg_set - key
						rest_set.foreach(rest_attr => {
							breakable {
								/* prevent special condition */
								/* if set{rest_attr} is already in tmp
								 * conflicts will occur.*/
								tmp.foreach(s => {
									if (s == mutable.Set(rest_attr)){
										break()
									}
								})
								tmp += (pos_set + rest_attr)
							}
						})
					}
				})

				new_pos_sets = tmp
			})
			(key, new_pos_sets)
		}.flatMap{ line =>
			line._2.map(s => (s, mutable.ArrayBuffer(line._1)))
		}.reduceByKey(_ ++ _)

		/**
		// collect the positive cover and save as file
		var pos_cover = pos_cover_dis.collect()

		// compensate for some cases like []: column i
		var idx_set: mutable.Set[Int] = mutable.Set()
		pos_cover.foreach(line => {
			idx_set ++= line._2.toSet
		})
		idx_set = R -- idx_set


		idx_set.foreach(i => {
			pos_cover :+= (mutable.Set[Int](), ArrayBuffer(i))
		})*/

		pos_cover_dis.map { (result) =>
			val left1 = result._1.toArray.sortBy(items => items).map(left => s"column$left").mkString("[",",","]")
			val right1 = result._2.sortBy(items => items).map(right => s"column$right").mkString("",",","")
			s"$left1:$right1"
		}.saveAsTextFile(resultPath)

	}

}
```


