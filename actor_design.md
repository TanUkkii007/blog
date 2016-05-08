# アクタープログラミングにおける設計について

アクターの設計は難しいとよく聞く。

## すべては単一責務原則


## インターフェースはメッセージ

アクターにおけるインターフェースはメッセージだ。アクター内部は完全にカプセル化されており、公開されているのはメッセージだけだからだ。アクター同士が連携をとる唯一の方法はメッセージパッシングだ。なのでメッセージの設計は非常に重要だ。オブジェクト指向や関数型プログラミングでインターフェースである公開関数のシグニチャを熟考するのと同様、アクタープログラミングではメッセージの設計を熟考する。

メッセージの設計におけるいくつかのアドバイスを書いておく。

## メッセージを汎化しすぎない


## メッセージを抽象化しない

ScalaにはOption[T]やTry


## メッセージを共有しすぎない


## メッセージに振る舞いを持たせない

このようなメッセージを見たことがある。メッセージが共通トレイトを継承してそのメソッドを実装し、アクター側で投げられるメッセージに応じて呼ぶ振る舞いを変えるというものだ。GoFデザインパターンにおけるテンプレートメソッドパターンをやりたかったのだろう。

```scala
trait Search {
  val text: String
  def search(): Future[SearchResult]
}

case class BingSearch(text: String) extends Search {
  def search() = doBingSearch(text)
}

case class GoogleSearch(text: String) extends Search {
  def search() = doGoogleSearch(text)
}
```

```scala
class SearchActor extends Actor {
  import context.dispatcher
  import akka.pattern.pipe
  
  def receive: Receive = {
    case msg: Search => msg.search().pipeTo(sender())
  }
}
```

このようなコードのモチベーションとしては、アクターのコードを共通化したかったのだろう。アクター内部のメソッドはカプセル化されているので、メッセージでしか外からコントロールできないからだ。

だがこのコードはお勧めしない。振る舞いはアクターの中にあるべきだ。まずメッセージはアクター間で共有されるので、カプセル化が破れる。この例だとメッセージを手に入れたアクターはだれでも検索エンジンにアクセスできてしまう。そして振る舞いには障害がつきまとう。アクターは障害に対処する必要があるので、振る舞いはアクターの中に閉じ込めよう。

## アクターは障害単位

そしてこの検索アクターは単一責務原則に違反している。アクターは障害単位だということを思い出して欲しい。この例だとGoogleとBingという外部サービスで起きる障害や制約は異なるだろう。例えば１秒間あたりのアクセス数制限やSLAはサービスによって異なるので、たとえSearchとSearchResultというメッセージのプロトコルが共通でも同じアクターにするのはよくない。

以下のコードではBingSearchActorとGoogleSearchActorに分た。そしてSearchActorという親アクターの子としてそれらを配置してある。またGoogleSearchだけ失敗しても３回までリトライを行っている。このようにアクターを障害単位という視点で分割すると独立に失敗でき、異なる耐障害戦略をとることができる。

```scala
class BingSearchActor extends Actor {
  import context.dispatcher
  import akka.pattern.pipe
  
  def receive: Receive = {
    case BingSearch(text) => doBingSearch(text).pipeTo(sender())
  }
}

object BingSearchActor {
  def props: Props = Props(new BingSearchActor)
}

class GoogleSearchActor extends Actor {
  import context.dispatcher
  import akka.pattern.pipe
  
  def receive: Receive = {
    case GoogleSearch(text) => doGoogleSearch(text).pipeTo(sender())
  }
  
  override def preRestart(reason: Throwable, message: Option[Any]): Unit = {
    message.foreach(self forward _)
    super.preRestart(reason, message)
  }
}

object GoogleSearchActor {
  def props: Props = Props(new GoogleSearchActor)
}

class SearchActor extends Actor {

  override def supervisorStrategy = OneForOneStrategy(maxNrOfRetries = 3) {
    case e: GoogleSearchException => Restart
  }
  
  val googleSearch = context.actorOf(GoogleSearchActor.props, "googleSearch")
  val bingSearch = context.actorOf(BingSearchActor.props, "bingSearch")
  
  def receive: Receive = {
    case msg: BingSearch => bingSearch forward msg
    case msg: GoogleSearch => googleSearch forward msg
  }
}
```


## アクターはコンポーザブル

アクターはコンポーザブルではないという意見を聞くことがあるが、それは間違いだ。
単一責務によく設計されたアクターはとてもコンポーザブルだ。

アクターの合成とは複数のアクターを組み合わせてより大きな機能を達成するアクターを作ることだ。合成はアクターヒエラルキーツリーの構築によって行う。

例としてまた検索アクターを使う。今回クライアントは検索エンジンに関心はなく、とにかく速く整形された検索結果が欲しいとしよう。これをアクターで実現するには一番早く検索できる検索エンジンから結果を取得してその結果を整形するということを１つのアクターでおこなうのではなく、１つのことに特化したアクターを組み合わせて最終的な結果を取得する調停役のアクターを作る。

```scala
class CompetitiveSearch(competiter: List[ActorRef], replyTo: ActorRef) extends Actor {

  def receive: Receive = {
    case msg: Search => competiter.foreach(_ ! msg)
    case msg: SearchResult =>
      replyTo ! msg
      context.stop(self)
  }
}

object CompetitiveSearch {
  def props(competiter: List[ActorRef], replyTo: ActorRef): Props = Props(new CompetitiveSearch(competiter, replyTo))
}

class SearchActor extends Actor {
  val googleSearch = context.actorOf(GoogleSearchActor.props, "googleSearch")
  val bingSearch = context.actorOf(BingSearchActor.props, "bingSearch")
  
  def receive: Receive = {
    case Search(text) => context.actorOf(CompetitiveSearch.props(List(googleSearch, bingSearch), sender()))
  }
}
```

アクターの合成のいいところは並列性、分散性、耐障害性を保ったまま合成できるということだ。アクターの合成は既存のアクター単位をそのまま使って組み立てるので、合成によって並列性が下がったり分散できなくなったり異なる耐障害コードが混入したりすることはない。アクターの良さを保ったままより大きなことが実現できる。


## per-request actorを好め

前のコードではCompetitiveSearchアクターをメッセージが来るたびに作成し、senderを渡してそのリクエスト専用に処理を行うアクターにした。なぜそうしたのだろう？リクエストごとにアクターを作るのは無駄なのではないかと思う人もいるだろう。アクターは複数のメッセージを受け取れるので、必ずしもリクエストごとに作る必要はないからね。

このようにリクエストごとに作られレスポンスを返すと寿命を終えるアクターはper-request actorと呼ばれる。per-request actorを選択できるときはそれを使うことをおすすめする。アクターを１つのリクエスト専用にすることで責務を限定でき、様々なメリットが得られるからだ。

まず１つのアクターであらゆるリクエストを捌くようにすると実装が難しくなる。CompetitiveSearchを終始メッセージを受け付ける長寿命アクターにすると実装はこうなるだろう。

```scala
class CompetitiveSearch(competiter: List[ActorRef]) extends Actor {
  var requestMap: Map[UUID, ActorRef] = Map()

  def receive: Receive = {
    case msg: Search => 
      requestMap += (msg.id -> sender())
      competiter.foreach(_ ! msg)
    case msg: SearchResult if requestMap.contains(msg.id) =>
      requestMap -= msg.id
      replyTo ! msg.result
  }
}
```

CompetitiveSearchは返信先の`replyTo: ActorRef`をキャプチャしなくなった代わりに内部に誰にどのレスポンスを返すべきか管理するMapを持たなければいけなくなった。またメッセージに識別子idをつけなくいけなくなり、インターフェースを変える必要が出てきてしまった。返信先をメッセージにのせるという方法もあるが、これもまたインターフェースを変える必要があり、かつ遅い方の検索結果が重複して届くということをクライアントが意識しなくてはいけなくなる。複数のアクターが関与しているとき誰が最終的に返答すべきかという問題はそれだけで語る価値があるので（ToBe）で触れよう。

次に耐障害性の喪失だ。あるリクエストによってCompetitiveSearchに障害が起きた場合、`requestMap`が失われ他のリクエストにも影響が出てしまう。クライアント間で独立に失敗できるように、異なるアクターで処理すべきだ。

そして並列性が下がっている。クライアントを順番に処理する必要はないだろう。アクターを分ければ並列に処理できる。アクターが並列単位だからね。

## アクターの寿命は短く


## アクターは惜しみなく作りまくれる

リクエストごとにアクターを作るのは無駄なのでパフォーマンスに悪影響なのではないかという疑問に対しての答えはNoだ。
１つのアクターサイズはたった[300byte](http://doc.akka.io/docs/akka/2.4.4/general/actor-systems.html#What_you_should_not_concern_yourself_with)だ。[Akka in Action](https://www.manning.com/books/akka-in-action)によると270万アクターを作っても1Gのメモリーしか消費しない。設計のシンプルさや耐障害性や並列性と引き換えにするには安すぎるオーバーヘッドだ。

アクターを作ることを躊躇しないでほしい。アクターインスタンスを作るオーバーヘッドは小さい。そしてインスタンスを作ってもメッセージを受け取るまでは何も実行されない。アクターはスレッドとは違い軽量なプロセスで、Akkaではただのオブジェクトだ。

## アクターを作ることを躊躇しない

アクターインスタンスを作ることを躊躇しないでほしいのと同じように、アクターの定義を作ることも躊躇しないでほしい。

チーム開発をしていて機能追加を頼まれるとついつい既存のアクターに手を加えて拡張してしまいがちだ。手を加える前に常に単一責務原則に基づき新しいアクターにすべきか考えて欲しい。機能を追加するということはあたらしく障害をもたらすので、それに対処する必要もある。手を加えるアクターのコードをよく把握し適切に拡張しなければならない。追加する機能がそれ単体で意味をなし、既存の機能とは独立に失敗すべきであるならば、新しいアクターを定義しよう。そのほうがクリーンな設計であり、チーム開発の分業としても合理的だ。
新しいアクターは独立にテストでき、既存の機能には左右されず、自分のコードに集中することができる。既存のアクターと連携するときはインターフェースであるメッセージだけ見ればよく、内部の実装は気にしなくて良い。新しいアクターが起こす障害も既存のアクターの挙動に影響を与えない。

チームの開発リーダーは単一責務原則に基づきアクターの単位を適切に判断し、メンバーを誘導するといいだろう。

## context.becomeの代わりにper request actorが使えないか検討する


## 監視の流れとメッセージフローを同じにする

メッセージフローの設計はアクターの設計の中でも難しいものの１つだ。


## 勝手に死ぬな


## アクターの寿命をマイクロマネージメントしない


## tell, don't ask, but acknowledge


## ヒエラルキーの頂点までAckする


## オブジェクト指向や関数型で考えない


## 長寿命状態アクターに挑戦する

これまでの