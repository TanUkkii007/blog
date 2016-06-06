# Akka Cluster

## メンバーシップ

各ノードは`hostname:port:uid`で識別される

`Join `コマンドをクラスターのメンバーのうち１つに送ることで参加する

AkkaはUIDをリモート死活監視のために使う。`uid`があるため、同じ`hostname:port`をもつActorSystemは一度クラスターから外されたら再参加することはできない。再参加するにはActorSystemを再起動して新しい`uid`を再発行する。

### Gossip

クラスターメンバーシップはAmazon Dynamoに基づき、特にRiakのアプローチを参考にしている。クラスターの状態はGossip Protocolで


### Vector Clocks

`(node, counter)`


複数ノード間でデータを同期するとき、並列書き込みと因果的依存書き込みを区別する必要がある。因果的依存書き込みは上書き可能だが、並列書き込みに対して上書きすると書き込みが失われる。
https://github.com/akka/akka/blob/master/akka-cluster/src/main/scala/akka/cluster/VectorClock.scala#L42-L45
並列書き込みは上書きせず、クライアント側でマージを行う必要がある。
Vector Clockはレプリケーションを実現するための技術の１つ。リーダーがいなくてかつ複数のレプリカがいる場合Vector Clockがひつようになる。
Nodeごとにバージョンが必要。書き込みのたびにバージョンを上げる。他のノードから読み込んだときは上書きするかマージするか決める。

Akka Clusterではメンバーの状態はCRDTで、自動マージが可能。


### Gossip Convergence

あるノードが見ている状態をすべてのノードが観測しているとき、Gossipは収束している。
Gossipの収束は`unreachable`なノードがある場合は行えない。`unreachable`なノードは`reachable`になるか`down`か`remove`になる必要がある。`unreachable`なノードがいる間、リーダーはリーダーアクションがとれないため、ノードの追加ができなくなる。

### Failure Detector

ノードが`unreachable`かどうか判断するのにFailure Detectorを使う。
accrual failure detectorは実測値と解釈を分離する。過去のハートビート履歴からの統計値で、ノードが生きているか死んでいるかの尤度を計算する。

`threshold`はユーザーが決められる値。低い`threshold`は誤った疑いをかけやすいが、素早い判断ができる。逆に高い`threshold`は誤った判断をしにくいが、実際のクラッシュを検出するのに時間がかかる。
デフォルトの値8は多くの場合に有効だが、Amazon EC2のようなクラウド環境ではネットワークの問題を考慮して12にしたほうがいいかもしれない。

モニタリングするノードはハッシュ順に並んだノードリングの近接ノードに対してなされる。
？？？

ハートビートは毎秒送られ、返答をFailure Detectorの入力に使う。

Failure Detectorはノードが再び`reachable`になることを判断するためにも使われる。

もしシステムメッセージがノードに送れなかった場合、ノードは隔離され`unreachable`から戻ることはできない。
これはwatchやTerminated、リモートアクターのデプロイ、リモートアクターによるスーパービジョンの際の障害通知のようなシステムメッセージのackがまりにも返ってこない場合に起きる。このときノードは`down`か`removed`状態になる必要がある。再度参加するにはActorSystemを再起動する。


### Leader

Gossipが収束したあと、リーダーが決定される。（Gossipの収束とはすべてのノードが同じ状態を観測すること）
リーダーを選出するというプロセスはない。Gossip収束時に決定論的にリーダーは認識される。
リーダーはただの役割で、あらゆるノードがリーダーになることができ、収束のラウンドごとに変わり得る。
リーダーはソートされたノードのうち最初のものである。

リーダーの役割はメンバーをクラスターから出入りするよう変えること。`joining`なメンバーを`up`状態にし、`exiting`なメンバーを`removed`にする。

設定すればリーダーはさらにノードをFailure Detectorに基いて"auto-down"することができる。これは`unreachable`なノードを指定時間後に`down`状態にすることである。


### Seed Nodes

シードノードは新しいノードがクラスターに参加するときのコンタクトポイント。
新しいノードが起動したらすべてのシードノードにメッセージを送り、最初に返答したシードノードにJoinコマンドを送る。

### Gossip Protocol

push-pull gossipはクラスターに送る情報量を削減するために使われる。push-pull gossipでは実際の値ではなくダイジェストを送る。gossipの受信者は新しいあらゆる値を返すことができ、古い値を持っていた場合は値を要求することができる。

Akkaはバージョニングに１つの共有状態としてVector Clockをもち（ClusterCoreDaemon）、push-pull gossipはpush時のみこのバージョンを使う。

- ClusterDaemon("clister")
|- ClusterHeartbeatReceiver("heartbeatReceiver")
|- ClusterCoreSuperVisor("core")
 |- ClusterCoreDaemon
 
周期的に（デフォルトでは１秒）各ノードはランダムにノードを選び、gossipのラウンドを開始する。

1/2以下のメンバーの状態しか見えていない場合、gossipの間隔を３倍にする。（ClusterCoreDaemon#isGossipSpeedupNeeded）収束の初期にこのスピードアップはおきる。

ノードの選択はランダムだが、まだ現在の状態をみていないノードにバイアスがかかる。バイアスの確率は0.8（gossip-different-view-probability）
https://github.com/akka/akka/blob/master/akka-cluster/src/main/scala/akka/cluster/ClusterDaemon.scala#L779-L787

このバイアス選択は収束プロセスの後半を加速させるため。

４００以上のクラスターでは、0.8という確率はgossipリクエストの集中による負荷を引き起こすため低く調整される（reduce-gossip-different-view-probability）。
https://github.com/akka/akka/blob/master/akka-cluster/src/main/scala/akka/cluster/ClusterDaemon.scala#L823-L828
gossipの受信者は同時にリクエストが集中するとメッセージをドロップする。？？？

クラスターが収束状態に入ったらgossip送信者はgosspiバージョンが入った小さなメッセージをノードに送る。クラスターの状態に変化があったら収束状態ではなくなり、バイアスのかかったgossipを再開する。

gossipの受信者はgossipのバージョン（Vector Clock）を使って
1. 新しいバージョンのgossip状態を持っていたら、gossip送信者に返す
2. 古いバージョンを持っていたら、今のバージョンとともに送信者に現在の状態を送るようリクエストする
3. 競合するgossipバージョンがあったら、マージして送り返す https://github.com/akka/akka/blob/master/akka-cluster/src/main/scala/akka/cluster/ClusterDaemon.scala#L693-L714


同じバージョンであれば何もしない。

gossipの周期的な特性は状態変化のバッチとしてよく働く？？？

gossipメッセージはprotobufでシリアライズされgzip圧縮して送られる。

### Membership Lifecycle

ノードは`joining`状態から始まる。すべてのノードが、そのノードがjoining状態であることを認識すると、`leader`はそのメンバーを`up`状態にする。

もしノードが安全で予期される方法でクラスターから外れる場合、そのノードは`leaving`状態になる。リーダーが、そのノードの`leaving`状態が収束したら、リーダーはそのノードを`exiting`状態にする。すべてのノードが`exiting`を観測したら（収束）、リーダーはそのノードをクラスターから取り除き、`removed`状態にする。

もしノードが`unreachable`であれば、gossipの収束はできなくなり`leader`アクションはとれなくなる（たとえばノードがクラスターに参加することはできない）。前に進むには`unreachable`状態を変えなければならない。`reachable`か`down`にならなければいけない。もし（`down`）からクラスターに参加したければ、ActorSystemを再起動して、参加プロセスを最初からやり直す。
リーダーを通して設定時間後にノードをauto-downすることもできる。もし`unreachable`ノードの生まれ変わりが再加入した場合古いものは`down`とされ、新しいのもは人手の介入なしに再加入できる。




