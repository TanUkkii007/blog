An Introduction to HMM-Based Speech Synthesis


# Chapter 1 The Hidden Markov Model

隠れマルコフモデルは時系列統計モデルの１つ。


## 1.1 Definition

HMMは離散的な時系列観測結果列を生成する有限状態機械である。各時間単位でHMMは状態遷移確率をもとにマルコフプロセスにおける状態を変えていき、観測データ$o$を現在の状態の出力確率分布とともに生成する。

N状態のHMMは状態遷移確率$A=a_{i,j}^N$、出力確率分布$B=b_i(o)_{i=1}^N$、初期状態確率$\Pi=[\pi_i]_{i=1}^N$で定義できる。簡易的にHMMのパラメーターを以下のように書く。

$\lambda = (A,B,\Pi)$

状態$i$の観測データ$o$の出力確率分布$b_i(o)$は連続でも離散でもよい。連続分布の場合、出力確率分布はよく多変数混合ガウス分布でモデリングする。

$b_i(o) = \sum_{m=1}^Mw_{im}N(o;\mu_{im},\Sigma_{im})$

$M$は分布の混合コンポーネントの数、$w_{im}$、$\mu_{im}$、$\sum_{im}$は重み、L次元平均ベクトル、$L \times L$次元の共分散行列。

各コンポーネントのガウス分布$N(o;\mu_{im},\Sigma_{im})$の定義は以下である。

$N(o;\mu_{im},\Sigma_{im}) = \frac{1}{\sqrt{(2\pi)^L|\Sigma_{ims}|}}exp(-\frac{1}{2}(o-\mu_{im})^\top\Sigma_{im}^{-1}(o-\mu_{im}))$

観測ベクトル$o_t$が互いに独立なデータストリームに分解できるなら、つまり$o = [o_1^\top, o_2^\top, ..., o_S^\top]^\top$なら、$b_i(o)$は混合ガウス分布の密度の積になる。

$b_i(o) = \prod_{s=1}^Sb_{is}(o_s)$
$ = \prod_{s=1}^S[\sum_{m=1}^{M_s}w_{ism}N(o_s;\mu_{ism},\Sigma_{ism})]$

$M_s$はストリームs中のコンポーネント数、$w_{ism}$、$\mu_{ism}$、$\sum_{ism}$はストリームsの状態iのコンポーネントmの重み、L次元平均ベクトル、$L \times L$次元の共分散行列。

### 1.1.1 Probability Evaluation
