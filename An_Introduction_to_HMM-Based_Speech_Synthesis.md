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

長さTの状態列が$q = (q_1,q_2,...q_T)$で決まるとき、HMM $\lambda$が与えられた時の長さTの観測列の観測確率$O = (o_1,o_2,...,o_T)$は各状態の出力確率をかけることで計算できる。

$P(O|q,\lambda) = \prod_{t=1}^TP(o_t|q_t,\lambda) = \prod_{t=1}^Tb_{qt}(o_t)$

状態列qの確率は状態遷移確率をかけることで計算できる。

$P(q|\lambda) = \prod_{t=1}^Ta_{q_{t-1}q_t}$

ここで$a_{q0i} = \pi_i$が初期状態の確率である。ベイズの定理により、Oとqの同時確率は

$P(O,q|\lambda) = P(O|q,\lambda)P(q|\lambda)$ //多変数における乗法定理を参照

HMM $\lambda$が与えられた時の観測列Oの確率は状態列qの周辺化、つまりすべてのqに対して$P(O,q|\lambda)$を足し合わせることで計算できる。

$P(O|\lambda) = \sum_{q}P(O,q|\lambda)$ //加法定理

$ = \sum_{q}P(O|q,\lambda)P(q|\lambda)$ //乗法定理

$ = \sum_{q}\prod_{t=1}^Ta_{q_{t-1}q_t}b_{q_t}(o_t)$ (1.13)*

*(この式は現実的に計算できない。計算量は$2TN^T$になる。各$q_1,...,q_T$にはN個のとりうる状態があり、$\sum_{q}$で$N^T$の計算量がある。さらに$\sigma$の中では$2T$個の掛け算がある。)

状態列が格子状**だと考えると、$\forall t \in [1,T]$に対して観測列の確率はこのように変換できる。

**(Figure1.1のような状態遷移図を時間方向に展開すると格子状になる。そのように図示したものをトレリス図という。)

$P(O|\lambda) = \sum_{i=1}^NP(O, q_t = i|\lambda)$ //加法定理

$P(O|\lambda) = \sum_{i=1}^NP(o_1,o_2,...,o_t, o_{t+1}, ..., o_T, q_t = i|\lambda)$

$P(O|\lambda) = \sum_{i=1}^NP(o_1,o_2,...,o_t,q_t=i|\lambda)P(o_{t+1},...,o_T|o_1,o_2,...,o_t,q_t=i,\lambda)$ //乗法定理

$P(O|\lambda) = \sum_{i=1}^NP(o_1,o_2,...,o_t,q_t=i|\lambda)P(o_{t+1},...,o_T|q_t=i,\lambda)$ //$o_{t+1},...,o_T$は$o_1,o_2,...,o_t$と独立

ゆえに観測列の確率(1.13)はフォワード・バックワード確率をつかって効率的に計算できる。

$\alpha_t(i) = P(o_1,o_2,...,o_t,q_t=i|\lambda)$

$\alpha_t(i)$は$\lambda$が与えられた時の状態i、時間tにおける$o_1,o_2,...,o_t$までの部分観測列の確率

$\beta_t(i) = P(o_{t+1},o_{t+2},...,o_T|q_t=i,\lambda)$

フォワード・バックワード確率は以下のように再帰的に計算できる

1 初期化

$\alpha_1(i) = \pi_ib_i(o_1)$ $1 \le i \le N$

$\beta_T(i) = 1$ $1 \le i \le N$

2 再帰

$\alpha_{t+1} = [\sum_{j=1}^{N}\alpha_t(j)a_{ij}]b_i(o_{t+1})$ $1 \le i \le N, t=2,...,T$

$\beta_t(i) = \sum_{j=1}^Na_{ij}b_j(o_{t+1})\beta_{t+1}(j)$$1 \le i \le N, t=T-1,...,1$

よって$\forall t \in [1,T]$に対して$P(O|\lambda)$は

$P(O|\lambda) = \sum_{i=1}^{N}\alpha_t(i)\beta_t(i)$

![フォワード確率の再帰的計算方法](https://github.com/TanUkkii007/blog/blob/master/img/HMM_forward_probability.png)

![バックワード確率の再帰的計算方法](https://github.com/TanUkkii007/blog/blob/master/img/HMM_backward_probability.png)

--------------------------------

ベイズの定理

$p(X,Y)$は同時確率で、「XかつYの確率」。$p(Y|X)$は条件付き確率で、「Xが与えられた下でのYの確率」。

XとYが同時に起きるということは、「Xがおきて、さらにXが起きた条件の下にYが起きる」ということだ。これを確率の乗法定理といい、$p(X,Y) = p(Y|X)p(X)$


乗法定理と対称性$p(X,Y) = p(Y,X)$により

$p(Y|X)p(X) = p(X|Y)p(Y)$

$p(Y|X) = \frac{p(X|Y)p(Y)}{p(x)}$

これをベイズの定理という。

加法定理$p(X) = \sum_Yp(X,Y)$を使うと、ベイズの定理は次のように表せる。

$p(Y|X) = \frac{p(X|Y)p(Y)}{\sum_Yp(X,Y)}$

$p(Y|X) = \frac{p(X|Y)p(Y)}{\sum_Yp(X|Y)p(Y)}$


--------------------------------

多変数における乗法定理

同時確率と条件付き確率は、変数が増えても同様に表せる。$P(X_1,X_2,...X_K)$はK個の確率変数の同時確率で、$P(X_1,X_2,...X_k|X_{k+1},X_{k+2},...X_{K})$は$X_{k+1},X_{k+2},...X_{K}$が与えられた時の$X_1,X_2,...X_k$の条件付き確率分布である。

複数の確率変数があった時の乗法定理を考える。式を見ると同時確率の引数のうち一部を条件部にもっていくことができる。

例えば$P(x_1,x_2,x_3,x_4,x_5) = P(x_1,x_2|x_3,x_4,x_5)p(x_3,x_4,x_5)$

また$P(x_1,x_2|x_3)$のように引数に条件がついても

$P(x_1,x_2|x_3) = P(x_1|x_2,x_3)P(x_2|x_3)$

が成り立つ。このとき$x_3$はもともとの前提として与えられているので、$P(x_1|x_2,x_3)$にも$P(x_2|x_3)$にも条件部に存在することに注意する。

この関係を使って繰り返し変形が可能である。

$P(x_1,x_2,x_3,x_4) = P(x_2,x_3,x_4|x_1)P(x_1)$
$ = P(x_3,x_4|x_1,x_2)P(x_2|x_1)P(x_1)$
$ = P(x_4|x_1,x_2,x_3)P(x_3|x_1,x_2)P(x_2|x_1)P(x_1)$

--------------------------------

確率の独立性

乗法定理ではXとYの同時確率は以下となった。

$p(X,Y) = p(Y|X)p(X)$


XとYが独立のとき、$P(Y|X) = P(Y)$となる。よって、

$p(X,Y) = p(Y)p(X)$


## 1.2 Optimal State Sequence

与えられた観測列$O = (o_1,o_2,...,o_T)$における最良の状態列$q^* = (q^*_1,q^*_2,...,q^*_T)$も様々なアプリケーションで有用である。例えば音声認識システムでは観測列ともっともとりうる状態列の同時確率を$P(O|\lambda)$を推定するためにつかう。

$P(O|\lambda) = \sum_qP(O,q|\lambda)$ //加法定理

$ \simeq max_qP(O,q|\lambda)$

最良の状態列$q^* = argmax_qP(O,q|\lambda)$はViterbiアルゴリズムとよばれる動的計画法に似た手法で計算できる。

$\delta_t(i)$を時間tのときに状態iで終わるもっともとりうる状態列とする。

$\delta_t(i) = max_{q_1,q_2,...,q_{t-1}}P(o_1,...,o_t,q_1,...,q_{t-1},q_t=i|\lambda)$

そして$\psi_t(i)$をそれを追跡する配列とする。これらを用いてViterbiアルゴリズムは以下のように書ける。

1 初期化

$\delta_1(i) = \pi_ib_i(o_1)$  $1 \le i \le N$

$\psi_1(i) = 0$  $1 \le i \le N$

2 再帰

$\delta_t(j) = max_i[\delta_t(i)a_{ij}]o_t$  $1 \le i \le N, t=2,...,T$

$\psi_t(j) = argmax_i[\delta_t(i)a_{ij}]$  $1 \le i \le N, t=2,...,T$

3 終了

$P(O,q^*|\lambda) = max_i[\delta_T(i)]$

$q^*_T = argmax_i[\delta_T(i)]$

4 パスの引き返し

$q^*_t = \psi_{t+1}(q^*_{t+1})$


## 1.3 Parameter Estimation

最尤推定(ML)のような、以下の式のようにある最適条件を満たすモデルパラメーターを解析的に求める方法はない。

$\lambda^* = argmax_{\lambda}P(O|\lambda)$
$ = argmax_{\lambda}\sum_{q}P(O,q|\lambda)$ //加法定理

この問題は隠れ変数qを含み不完全なデータからの最適化問題のため、与えられた観測列Oのための尤度P(O|\lambda)をグローバルに最大化する$\lambda^*$を閉形式で求めるのは難しい。

しかしながらローカルに$P(O|\lambda)$を最大化するパラメーター$\lambda$を得るのは完全なデータセットの最適化を行えるEMアルゴリズムのような手法で得られる。この最適化アルゴリズムはよくBaum-Welchアルゴリズムと呼ばれる。

以下の節でガウス分布を用いたCD-HMMのEMアルゴリズムを解説する。離散分布や混合ガウス分布のEMアルゴリズムも自明に導かれる。

### 1.3.1 Auxiliary Function Q

