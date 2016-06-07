# Akkaはマルチスレッドプログラミングにおける可視性の問題をどう解決しているか

アクターのロケーション透過性は、アクターがどのサーバー、どのCPUコア、どのスレッド上で実行されようが同じ挙動になることを保証する。これはプログラマがアクターの実行環境を意識しなくていいという点ですばらしい。この性質により僕のような並列プログラミングや分散システムにそんなに詳しくない開発者でも安全に並列分散システムのコードを書くことができている。

アクターが並列プログラミングを簡単にするもう１つの特性はカプセル化である。アクターは内部にミュータブルな状態をもつことができ、それを安全に更新することができる。つまりvarを使うことができる。ここでもプログラマは並列環境で実行されていることを意識する必要はない。アクターの強いカプセル化は内部状態を外からのアクセスから守ってくれる。

もしプログラマが並列環境を意識しなくてはいけなかったら、このアクター内のvarは@volatile宣言がされてなければいけない。なぜならあるスレッドでのvarの変更は別のスレッドで見えているとは限らないからだ。varの値の変更があるCPU上のスレッドでなされた場合、その変更を別のCPUが観測するにはそのCPUのローカルキャッシュを更新する必要がある。これにはしばらく時間がかかる。volatile変数の場合、他のスレッド書き込んだ変数の値は必ず他のスレッドにも伝播しているよう保証してくれる。

ここでの疑問は、プログラマが@volatile宣言をしなくてもいいようにAkkaがどのような工夫をしているかということだ。Akkaがロケーション透過性を実現するには、アクター内部の@volatile宣言のないvarの書き込みが次のメッセージを受信したときの実行スレッドで観測できている必要がある。

その答えは[StackOverflow](http://stackoverflow.com/questions/15849366/how-does-akka-implement-the-jmm-like-happens-before-relationship)でAkkaのコントリビューターであるRoland Kuhnさんが答えていた。

Akkaの[Java Memory Model](http://doc.akka.io/docs/akka/2.4.7/general/jmm.html#Actors_and_the_Java_Memory_Model)のドキュメントにあるように、Akkaは可視性と順序問題に対し２つの"happens before"ルールを保証しています。

1. The actor send rule: アクターへのメッセージ送信は、そのアクターがそのメッセージを受信する前に起きる
1. The actor subsequent processing rule: アクターでのあるメッセージの処理は、そのアクターでの次のメッセージの処理の前に起こる

この２つのルールを保証しているコンポーネントはメールボックスだ。

１つ目のルールは、メールボックスの`MessageQueue`の実装によって達成できる。アンバウンデッドメールボックスの場合`ConcurrentLinkedQueue`を使うことでvolatile writeを行い、バウンデッドメールボックスの場合は`LinkedBlockingQueue`を使うことでロックを行う。これによってメッセージの送信は、そのメッセージの処理の前に起きていることが観測できる。

２つ目のルールは、メールボックスのステータス管理によって実現している。メッセージを処理し終えたあと、メールボックスはアイドル状態になる[Mailbox.scala#L227](https://github.com/akka/akka/blob/v2.4.7/akka-actor/src/main/scala/akka/dispatch/Mailbox.scala#L227)。これは`sun.misc.Unsafe.compareAndSwapInt`を使っており、volatile writeである[Mailbox.scala#L129-L130](https://github.com/akka/akka/blob/v2.4.7/akka-actor/src/main/scala/akka/dispatch/Mailbox.scala#L129-L130)。メッセージを処理し始める時、メールボックスのステータスを確認する[Mailbox.scala#L222](https://github.com/akka/akka/blob/v2.4.7/akka-actor/src/main/scala/akka/dispatch/Mailbox.scala#L222)。これは内部で`sun.misc.Unsafe.getIntVolatile`を行っており、volatile readである[Mailbox.scala#L111](https://github.com/akka/akka/blob/v2.4.7/akka-actor/src/main/scala/akka/dispatch/Mailbox.scala#L111)。これにより以前のメッセージ処理時に行った書き込みは同期され、次のメッセージを処理するときに観測できることが保証される。

僕は@yoskhdiaさんに質問されるまで@volatile