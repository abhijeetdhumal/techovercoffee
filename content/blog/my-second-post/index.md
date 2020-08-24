---
title: My Second Post!
date: "2020-08-23T23:46:37.121Z"
slug: /my-second-post/
show: false
author: abhi
tags: 
    - Second
    - code
---

I never knew that blogging with gatsby will be so simple. 

## Word count in scala

```
val textFile = sc.textFile("hdfs://...")
val counts = textFile.flatMap(line => line.split(" "))
                 .map(word => (word, 1))
                 .reduceByKey(_ + _)
counts.saveAsTextFile("hdfs://...")
```
### Big Data Heading 2 
#### Big Data Heading 3 
##### Big Data Heading 4

