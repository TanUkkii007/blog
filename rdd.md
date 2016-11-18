Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing

https://www2.eecs.berkeley.edu/Pubs/TechRpts/2011/EECS-2011-82.pdf

# Abstract

Resilient Distributed Datasets (RDD)という、MapReduceのように耐障害性をもつインメモリクラスターコンピューティングのための分散メモリ抽象化を紹介する。RDDは既存のシステムでは非効率な２つの処理を動機にしている。機械学習のような繰り返しのアルゴリズムと、対話式のデータマイニングである。どちらの処理も、データをメモリ上にもつことは飛躍的なパフォーマンスの向上につながる。耐障害性を効率的に実現させるために、RDDはかなり制限された共有メモリを提供する。共有メモリは読み込みのみ可能であり、他のRDDからのバルク処理によってのみ構築できる。それでもRDDの表現力は非常に高いことを示す。RDDの実装は繰り返し処理ではHadoopの２０倍の性能であり、対話式の検索では１TBのデータを5−7sのレイテンシで実現する。

# 1 Introduction