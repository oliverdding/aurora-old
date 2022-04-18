+++
title = "到底什么是Spark Job"
description = "Spark的运行拆分为Job、Stage、Task，本文转载Stackoverflow比较好的一个解释"
date = 2022-04-15T15:15:43+08:00

[taxonomies]
categories = []
tags = ["spark", "big data", "reproduction"]

[extra]
toc = true
comments = true
+++

Well, terminology can always be difficult since it depends on context. In many cases, it can be used to "submit a job to a cluster", which for spark would be to submit a driver program.

That said, Spark has its own definition for "job", directly from the glossary:

> Job A parallel computation consisting of multiple tasks that gets spawned in response to a Spark action (e.g. save, collect); you'll see this term used in the driver's logs.

So I this context, let's say you need to do the following:

1. Load a file with people names and addresses into `RDD1`
2. Load a file with people names and phones into `RDD2`
3. Join `RDD1` and `RDD2` by name, to get `RDD3`
4. Map on `RDD3` to get a nice HTML presentation card for each person as `RDD4`
5. .Save `RDD4` to file.
6. Map `RDD1` to extract zipcodes from the addresses to get RDD5
7. Aggregate on `RDD5` to get a count of how many people live on each zipcode as `RDD6`
8. Collect `RDD6` and prints these stats to the stdout.

So,

1. The driver program is this entire piece of code, running all 8 steps.
2. Producing the entire HTML card set on step 5 is a job (clear because we are using the save action, not a transformation). Same with the collect on step 8
3. Other steps will be organized into stages, with each job being the result of a sequence of stages. For simple things a job can have a single stage, but the need to repartition data (for instance, the join on step 3) or anything that breaks the locality of the data usually causes more stages to appear. You can think of stages as computations that produce intermediate results, which can in fact be persisted. For instance, we can persist RDD1 since we'll be using it more than once, avoiding recomputation.
4. All 3 above basically talk about how the logic of a given algorithm will be broken. In contrast, a task is a particular piece of data that will go through a given stage, on a given executor.

---

原文：[What is Spark Job ?](https://stackoverflow.com/a/28974346)
