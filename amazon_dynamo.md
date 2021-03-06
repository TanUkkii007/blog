Dynamo: Amazon’s Highly Available Key-value Store

http://www.allthingsdistributed.com/2007/10/amazons_dynamo.html

# Abstract

大規模スケールにおいての信頼性はAmazon.com（もっとも大きいeコマースサービス）で直面した大きな課題の１つである。わずかなダウンでも金銭的に大きな結果を引き起こし顧客の信用を低下させる。世界中にWebサービスを提供するAmazon.comプラットフォームは複数のデータセンターの数万のサーバーとネットワークコンポーネントの上に実装されている。このスケールでは、大小のコンポーネントが常に障害を起こし続け、このような障害のもとでどのように永続的な状態を扱うかという手法はソフトウェアシステムに信頼性とスケーラビリティをもたらしてきた。

この論文では、Dynamoのデザインと実装を紹介する。Dynamoは高可用性のあるkey-valueストアで、Amazonの常に利用可能であるという体験を提供しているコアサービスである。このレベルの可用性を実現するには、Dynamoはいくつかの障害の状況で一貫性を犠牲にしている。オブジェクトのバージョニングとアプリケーションが引き起こしたコンフリクトの解決を、開発者が利用しやすい形でふんだんに用いている。

# 1. Introduction

Amazonは世界中のデータセンターにある何十万ものサーバーを使って１０００万ユーザーに提供するeコマースサービスを世界的に展開している。Amazonではプラットフォームがスケールするためにパフォーマンス、信頼性、効率性、に関する厳格なオペレーションの要求がある。なかでも信頼性はもっとも重要な要求の１つである。僅かなダウンが財政と顧客の信頼に影響を与える。

Amazonプラットフォームを運用して学んだ教訓の１つに、信頼性とスケーラビリティはアプリケーションの状態をどのように管理するかにかかっているということがある。Amazonは非中央集権的で疎結合であり、数百のサーバーからなるSOAをつかっている。このような環境では常に利用可能なストレージが必要になる。例えば顧客はディスクが故障していようと、ネットワークが混雑していようと、データセンターが竜巻で破壊されたとしても、ショッピングカートを見たり商品を追加できるべきである。ゆえに、ショッピングカートを担うサービスはデータストアから常に読み書きできればならず、データは複数のデータセンターに存在しなければならない。

百万のコンポーネントからなるインフラストラクチャで障害に対処することは私達のオペレーションの標準となっている。常に小さいながらも莫大な数のサーバーとネットワークコンポーネントがある時間に障害を起こしている。なのでAmazonのソフトウェアシステムは可用性やパフォーマンスに影響をあたえることなく障害を通常のケースとして扱う必要がある。

信頼性とスケーラビリティの要求を叶えるには、Amazonは複数のストレージ技術を開発した。S3が最もよく知られているだろう。この論文ではもう１つの高可用性かつ高スケール性の分散データストアであるDynamoのデザインと実装を紹介する。Dynamoは高信頼性をもちつつ、可用性と整合性、効率性とパフォーマンスのトレードオフが可能なサービスの状態を管理するために使われる。Amazonプラットフォームには異なるストレージ要求の多様なアプリケーションがある。いくつかのアプリケーションはアプリケーションデザイナに高可用性とコスト効率の高いパフォーマンスのトレードオフをもとにデータストアを設定することを求める。

Amazonプラットフォームにはデータストアに主キーのみのデータアクセスしか必要ないサービスがたくさんある。ベストセラーのリスト、ショッピングカート、顧客の好み、セッション管理、セールスランク、商品カタログなどのサービスでは、RDBを使うという一般的なパターンは非効率でスケーラビリティと可用性を制限する。Dynamoはこれらのアプリケーションの要求を満たすシンプルな主キーのみのインターフェースを提供する。

Dynamoはスケーラビリティと可用性を達成するためによく知られた技術を組み合わせている。データはconsistent hashingを使ってパーティションされレプリケーションされる。整合性はオブジェクトのバージョニングで実現する。更新時のレプリカの整合性はクォラムと非中央集権的レプリカ同期技術で担保する。Dynamoはgossipベースのfailure detectionとメンバープロトコルを採用した。Dynamoは最小限の管理ですむ完全に非中央集権的である。ストレージノードは手動のパーティショニングや再分散を必要とせずDynamoから追加削除できる。

過去何年もDynamoはAmazonのいくつかのコアサービスのストレージ技術となってきた。休暇のショッピングシーズンでもダウンタイムなくピークの負荷にスケールできた。例えば、ショッピングカートを担うサービスは１日に３００万の会計に相当する１０００万のリクエストをさばき、セッションサービスは数１０万の同時セッションの状態管理をさばいた。

研究コミュニティの主な貢献は１つの高可用性システムをつくるために異なる技術をどう組み合わせるか評価したことだ。結果整合なストレージシステムは厳しいアプリケーションでもプロダクションで使えることを証明した。プロダクションの厳しいパフォーマンス要求にこたえるためこれらの技術をチューニングする知見も提供した。

この論文は以下のように構成される。２章は背景を説明し、３章は関連する研究を紹介する。４章はシステムデザインを、５章は実装を解説する。６章はプロダクションでのDynamoの運用から得た経験と示唆を詳説し、７章では結論を述べる。この論文の随所でより詳しい解説が適切な場面があるが、Amazonのビジネスを守るために情報を絞った。そのため６章のデータセンター間、内、のレイテンシや、６.２章の絶対リクエスト数やダウンタイム、6.３章の負荷は絶対値ではなく平均を提供している。

# 2. Background

Amazonのeコマースプラットフォームはレコメンドサービスや決済、違反検知などの機能を提供するために協調する数百のサーバーから構成されている。各サービスはよく定義されたインターフェースで公開されており、ネットワークを介してアクセスできる。これらのサービスは世界中に分布する複数のデータセンターにある数万のサーバーからなるインフラストラクチャでホストされている。いくつかのサービスはステートレス（i.e. 他のサービスからのレスポンスを集積するサービス）で、いくつかはステートフルである（i.e. 永続ストアの状態を元にビジネスロジックを実行しレスポンスを生成するサービス）。

伝統的にプロダクションのシステムは状態をリレーショナルデータベースに保存していた。しかしながら状態の永続化のよくあるパターンとしてリレーショナルデータベースは理想には程遠い解決策だ。ほとんどのサービスはデータを主キーのみで保存、取得し、RDBMSの複雑なクエリや管理機能は必要ない。この過剰な機能は高価なハードウェアとオペレーションに高度なスキルをもつ人材が必要になり、非効率な解決策だ。加えて、利用可能なレプリケーション技術は限られており大抵可用性よりも整合性の方を選択している。ここ数年で進歩はしたものの、データベースをスケールアウトしたりロードバランスのための賢いパーティショニングはまだまだ簡単ではない。

この論文ではこれらの重要なサービスに言及する高可用性のあるデータベース技術であるDynamoを解説する。Dynamoは単純なkey/valueインターフェースで、明確に定義された整合性ウィンドウを持ちながら高可用であり、効率的なリソース使用で、データサイズやリクエストレートの上昇に対するシンプルなスケールアウト企画を有する。Dynamoを使う各サービスはDynamoインスタンスを実行している。

## 2.1 System Assumptions and Requirements

このクラスのサービスのストレージは以下の要求をもつ

- Query Model: 主キーによって識別できるread/write。状態はバイナリオブジェクト（blob）として保存される。複数のデータにまたがる操作はなく、スキーマにリレーションはない。この要求はAmazonのサービスのかなりの部分の調査に基づく。小さなオブジェクト（< 1MB）をターゲットにしている。
- ACID Properties: ACIDはトランザクションの信頼性を保証する性質。データベースでは１つの論理操作に相当。ACIDは可用性に乏しいことが証明されている。Dynamoは可用性の向上が望めるなら弱い整合性を認めるアプリケーションをターゲットとする。
- Efficiency: システムはコモディティなハードウェアで動作する必要がある。Amazonでは99.9パーセンタイルで測る厳しいレイテンシの基準がある。パフォーマンス、コスト効率、可用性、耐久性をユーザーが調節可能であるべき。
- Other Assumptions: 内部システムのためセキュリティは考慮しない

## 2.2 Service Level Agreements (SLA)

Amazonのような非中央集権なSOAでは、SLAは非常に重要である。例えばe-コマースサイトの１つへのページリクエストはページをレンダリングするのに１５０異常のリクエストを伴ってレスポンスを構築する。各サービスがパフォーマンスの契約に従わなければページレンダリングエンジンの品質の保証ができない。

Fig.1はAmazon Platformの抽象的な図である。サービスは異なるストレージを使い、サービス境界内のみからアクセス可能である。幾つかのサービスは複数のサービスを集約する。集約サービスはたいていステートレスだが、大規模なキャッシュを用いている。

すべての顧客に良い体験を提供するため、SLAに平均や中央値は用いず、99.9パーセンタイルを用いる。99.9パーセンタイルという数字はコスト・利益解析をもとに、これ以上のパフォーマンスの向上は著しいコストの上昇を伴う値を選択した。

ストレージシステムはSLAに重要な役割を果たす。とりわけビジネスロジックが比較的軽量な場合にそうであり、Amazonの多くのサービスがそれに当てはまる。Dynamoはサービスが耐久性、整合性、機能、パフォーマンス、コストのトレードオフをコントロール可能なように設計されている。

## 2.3 Design Considerations

高い整合性を実現するために伝統的なレプリケーションは同期的に行われてきた。しかしネットワーク故障のもとでは可用性と整合性は両立できない。可用性は非同期で、並列で、分断耐性のある楽観的なレプリケーション技術で向上できる。このアプローチはコンフリクトの検出と解決を行わなければならない点が難しい。いつ誰がコンフリクトを解決するかという２つの観点がある。

いつコンフリクトを解決するかというデザインは重要である。いつとは書き込み時か、あるいは読み込み時かである。伝統的なデータストアは書き込み時に解決する。それに対して、Dynamoは読み込み時に解決を行い、書き込み時にリクエストを拒絶したりはしない。

誰が解決するかという問題は、データストアかアプリケーションになる。データストアが解決する場合、last write winという単純な選択しかない。しかしアプリケーションが解決するとなれば、多様な解決方法が可能である。

他の主要なデザイン原理は、
- Incremental scalability
- Symmetry
- Decentralization
- Heterogeneity


# 3. RELATED WORK

## 3.1 Peer to Peer Systems

第一世代。構造化されていないP2Pネットワーク。ピアと任意のリンクを張る。

- Freenet
- Gnutella

有界のクエリーのホップ数

- Pastry
- Chord

O1ルーティング。ローカルにルーティング情報をもつ。

- Oceanstore
- PAST


## 3.2 Distributed File Systems and Databases

整合性より可用性をとった分散ファイルシステム

- Ficus
- Coda

中央サーバーのない分散ファイルシステム

- Farsite

マスターをもつシンプルな分散ファイルシステム

- Google File System

結果整合のRDB

- Bayou

このなかでBayou、Coda、Ficusは分断されたオペレーションに対応している。Coda、Ficusはシステムレベルのコンフリクト解決を行い、Bayouはアプリケーションレベルのコンフリクト解決を行う。

Antiquityはセキュアログとビザンチン故障耐性プロトコルを使ったデータ完全性をもつWANストレージシステム。Dynamoにはセキュリティの機能はない。

Bigtableは疎な多次元ソート済みマップを扱うストレージで、複数の属性のアクセスを可能にする。Dynamoはkey-valueアクセスしかない。

## 3.3 Discussion

上記の分散ストレージとは以下の点で異なる。

- 常に書き込み可能である
- 信頼できるネットワーク環境を想定する
- key-valueアクセスのみである
- readとwriteのレイテンシが99.9パーセンタイルで数百msを要求する。そのためルーティングはゼロホップDHTである。


# 4. SYSTEM ARCHITECTURE

以下を解説する。

- パーティショニング
- レプリケーション
- バージョニング
- メンバーシップ
- 障害ハンドリング
- スケーリング

表１に使用した技術とその利点を挙げる。

| 問題 | 技術 | 利点 |
|:----------:|:-----------:|:------------:|
| パーティショニング       |        Consistent Hashing |     インクリメンタルなスケーラビリティ     |
| 書き込み時の可用性     |  Vector clockと読み込み時の解決 |    バージョンサイズが更新頻度に依存しない    |
| 一時的な障害への対処 | Sloppy Quorumとhinted handoff | レプリカの一部がいなくても高い可用性と耐久性を提供|
| 恒久的な障害への対処 | Merkle treeによるアンチエントロピー | 発散したレプリカをバックグラウンドで同期する |
| メンバーシップと障害検知| Gossipベース | 対称性をたもち、メンバーシップとノードの生存情報を中央集約することを避ける |


## 4.1 System Interface

get()とput()の２つのインターフェイスがある。get(key)は１つの値あるいはコンフリクトした複数の値とcontextを返す。put(key, context, object)はkeyをもとにobjectを配置するレプリカを決め、永続化する。contextはobjectのバージョンなどのシステムのメタデータを有する。

keyとobjectはバイト配列で扱う。keyはMD5ハッシュを適用し128bitの識別子を生成し、保存するノードを決めるのに使われる。

## 4.2 Partitioning Algorithm

Dynamoのパーティショニングはconsistent hashingに基づいている。consistent hashingではハッシュ関数の出力領域は固定の循環空間つまりリングとして扱われる。各ノードは空間内のランダムな値を与えられ、リング上のある位置に配置される。データはキーのハッシュからリング上の位置を算出し、時計回りに歩き最初に行き当たるノードに配置される。consistent hashingの利点はノードの追加と削除は最も近所のノードにしか影響を与えず、他のノードには何も影響がないところだ。

基本的なconsistent hashingのアルゴリズムだと２つの欠点がある。

- ノードのリングへのランダムな配置はデータと負荷の分布を一様に分散しない
- ノードの不均一な性能を加味できない

そのためDynamoはconsistent hashingの派生アルゴリズムを用いている。ノードはリング状の複数位置に配置される。Dynamoはvirtual nodeという概念を導入している。パーティショニングのチューニングプロセスは６章で説明する。

virtual nodeの利点は以下である。

- あるノードが利用不可能担った場合、そのノードの負荷は他のノードに一様に分散される
- ノードが再び利用可能になったり追加された場合、他のノードとほぼ同等の負荷を担当する
- ノードが担当するvirtual nodeの数はそのノードの性能をもとに決められる。性能は不均一で良い。

## 4.3 Replication

各データはNホストにレプリケーションされる。Nはインスタンスごとに決められる値である。キーkはコーディネーターノードに担当される。コーディネーターはリングの範囲内のデータのレプリケーションに責任を持つ。リングの範囲内の各キーをローカルに保存することに加えて、コーディネーターは時計回りにN-1個のノードにレプリケーションする。つまり各ノードはそのノードとそれからN番目のノードの間のリングの領域を担当するということである。

あるキーを保存する責任があるノードのリストはpreference listと呼ばれる。ノードの故障を考慮すると、preference listはN以上のノードをもってないといけない。virtual nodeを使用する場合、あるキーのNノードはN台の物理ノードより小さい可能性があることに注意する。これに対処するため、preference listのノードは物理ノードを重複して保持しないようにスキップする。

## 4.4 Data Versioning



