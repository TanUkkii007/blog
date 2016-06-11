# The Google File System


## Abstract

負荷を計測するにつれ、伝統的な分散ファイルシステムとはかなり異なるデザインになった

## 1. INTRODUCTION

今と将来の予測に基づくアプリケーションの負荷と技術的環境によって、ファイルシステムのデザインは変わっていった。

- コンポーネントの障害は例外ではなく通常
- 巨大なファイル。数Gbが普通。全データでTbあるデータを数Kb単位で１０億管理するのは難しい
- 書き込みは追記。ランダム書き込みはしない。
- アプリケーションとファイルシステムAPIを一緒にデザインすることで、柔軟性やシンプルさを得る

複数のクラスター
大きい物は１０００台のノードで300Tb
１００台のクライアントアクセス


## 2. DESIGN OVERVIEW

### 2.1 Assumptions

仮定

- 常に障害の可能性をもつコモディティーハードウェア
- 適度な数の巨大なファイル。100Mbぐらいのファイル数百万。数Gbは普通。
- 負荷は主に２つのRead：大きなストリームReadと小さなランダムRead。
- 負荷は大きなシーケンシャルWriteもある。一度書いたファイルはほとんど変更されない。
- 並列に同じファイルに追記書き込みを行う複数のクライアントのよく定義されたセマンティックスを効率よく実装。ファイルは多様な方法でマージ可能なproduce-consumerキューとして使われる。数百のproducerをさばける。最小の同期オーバーヘッドのアトミック性が重要。consumerはファイルを後で読むか、同時に読み込む。
- 高い帯域は低いレイテンシよりも重要。我々のアプリケーションはデータをバルクで処理する

### 2.2 Interface

- reate
- delete
- open
- close
- read
- write
- snapshot
- record append (アトミック性を保ちながら複数のクライアントが同じファイルに追記する操作。multi-way mergeやproduce-consumerキューを実装するときに使う)

### 2.3 Architecture

１つのmasterと複数のchunkserverから成る。

ファイルは固定長のchunkに分割される。chunkはイミュータブルでグローバルに一意な64bitのchunk handleで識別される。ChunkserverはchunkをLinuxファイルとしてローカルのディスクに保存し、chunk handleとbyte rangeを与えられて読み書きする。chunkは複数のchunkserverにレプリケートされる。

masterはすべてのファイルシステムのメタデータを管理する。たとえばnamespace, access control, mapping from files to chunks, current location of chunksを管理する。
chunk lease management, 迷子chunkのガーベッジコレクション、chunkserver間のchunkのマイグレーションなどシステムワイドなアクティビティも管理する。masterはchunkserverと定期的にHeartBeatメッセージを送って命令を送ったり状態を収集したりする。

クライアントはメタデータ操作はmasterとやりとりするが、データは直接chunkserverとやりとりする。POSIX APIは実装していないのでLinuxのvnodeレイヤーはフックする必要がない。

クライアントもchunkserverもファイルデータをキャッシュしない。クライアントやシステムのキャッシュを実装しないことによって、キャッシュのコヒーレンス問題を考える必要がなく単純化できる（ただしクライアントはメタデータをキャッシュしている）。chunkはローカルのファイルとして保存されており、アクセス頻度の高いものはLinuxのバッファーキャッシュでメモリーに保持されるので、chunkserverはキャッシュを実装する必要はない。

### 2.4 Single Master


