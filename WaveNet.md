WAVENET: A GENERATIVE MODEL FOR RAW AUDIO


# Abstract

WaveNetという音響波形を生成するDNNを紹介する。このモデルは完全に確率論的で自己回帰的である。各音響サンプルは以前のすべてのサンプルで条件付けされる推定分布を用いる。それでも１秒あたり数万のサンプルを効率的に学習できた。英語と中国語のTTSに応用すると、歴代最高の品質となった。１つのWaveNetで複数の話者の特徴を捉えることができ、話者で条件付けするとそれに切り替えることができる。音楽モデルを学習した場合、現実的な音楽の断片を出力した。識別モデルにも使うことができ、音素認識に期待することができる。

# 1 INTRODUCTION

 この研究は画像(van den Oord et al., 2016a;b)、文章(Jo ́zefowicz et al., 2016)などの複雑な分布の自己回帰生成モデルにインスパイアされている。
 
 これらのアーキテクチャは数千の変数をモデルしており（PixelRNNの場合64*64）、音響波形のような1600samples/secの解像度に対してもこのアプローチが成功するのかを検証した。
 
 WaveNetはPixelCNNアーキテクチャに基づく音響生成モデルである。
 - 従来のTTSの分野では報告のないぐらい自然な音声を生成する
 - 音響生成に必要な長期間の依存性に対処するため、広い受容野をもつことで知られるdilated causal convolutionベースのアーキテクチャを開発した
 - 話者で条件付けすると、１つのモデルで複数の声を生成できる
 - 音声認識にも使うことができ、音楽も生成できる


# 2 WAVENET

波形$x = \{ x_1, ...,x_T \}$の同時確率はは条件付き確率をかけ合わせたものである。

$p(x) = \prod_{t=1}^Tp(x_t|x_1,...,x_{t-1})$

つまり各音響サンプルは以前のタイムステップのすべてのサンプルによって条件付けられている。

PixelCNNと同じように、条件付き確率分布は畳み込み層の積み重ねでモデリングされる。プーリング層はなく、出力層は入力層と同じ次元を持つ。ソフトマックス層で次の値$x_t$のカテゴリ分布を出力し、それはパラメーターに対してデータの対数尤度が最大化されるように最適化される。

## 2.1 DILATED CAUSAL CONVOLUTIONS

WaveNetのメインはcausal convolutionである。causal convolutionを使うことで、モデルがデータモデルの順序に違反しないことを保証できる。時間tのときにモデルが出力した予測$p(x_{t+1}|x_1,...,x_t)$は未来の値$x_{t+1}, x_{t+2}, x_t$に依存してはならない。画像での同等のcausal convolutionはmasked convolutionで、mask tensorを構築し、maskとconvolution kernelとの要素方向の掛け算を行う。音響データのような１次元の場合だと、通常のconvolutionの出力をタイムステップ分シフトして実装できる。

学習時には、すべてのタイムステップの条件推定の計算は並列に実行できる。モデルを使って生成するときは、予測は順次的に行う。各サンプルが予測されたら、次のサンプルにフィードバックされる。

causal convolutionは再帰的コネクションを持っていないため、RNNよりも学習が速い。特に長いシーケンスに適用した場合にそうだ。causal convolutionの問題の１つはたくさんの層、あるいは受容野を広げるための巨大なフィルターが必要なことである。例えばFig.2では受容野は５(層の数＋フィルターの長さ−１)しかない。この論文では受容野を広げるためdilated convolutionを使う。

dilated convolution（あるいは a` trous、convolution with holes）は入力を何ステップかスキップして、フィルターをそれよりも長い領域に適用するものである。これはゼロパディングしてより大きなフィルターを適用しconvolutionと同じだが、はるかに効率的だ。dilated convolutionはより素なネットワークを可能にする。プーリングやstrided convolutionと同じだが、出力が入力と同じ長さになる点が異なる。過去にdilated convolutionはシグナルプロセッシングや画像セグメンテーションに使われてきた。

stacked dilated convolutionは少ない層数で大きな受容野をもつことを可能にする。この論文ではdilationは各レイヤーで倍々にしていった。この理由は、
1. 指数関数的にdilation factorを増加させることは、深くなる毎に受容野が指数関数的に成長する(Yu & Koltun, 2016)
2. これらのブロックをさらに積み上げるとモデルのキャパシティが増大し大きい受容野につながる


## 2.2 SOFTMAX DISTRIBUTIONS





















