An Introduction to HMM-Based Speech Synthesis

Junichi Yamagishi


# Chapter 1 The Hidden Markov Model

隠れマルコフモデルは時系列統計モデルの１つ。


## 1.1 Definition

HMMは離散的な時系列観測結果列を生成する有限状態機械である。各時間単位でHMMは状態遷移確率をもとにマルコフプロセスにおける状態を変えていき、観測データ$o$を現在の状態の出力確率分布とともに生成する。

N状態のHMMは状態遷移確率$A=a_{i,j}^N$、出力確率分布$B=b_i(o)_{i=1}^N$、初期状態確率$\Pi=[\pi_i]_{i=1}^N$で定義できる。簡易的にHMMのパラメーターを以下のように書く。

$\lambda = (A,B,\Pi)$

状態$i$の観測データ$o$の出力確率分布$b_i(o)$は連続でも離散でもよい。連続分布の場合、出力確率分布はよく多変数混合ガウス分布でモデリングする。

$b_i(o) = \sum_{m=1}^Mw_{im}\mathcal{N}(o;\mu_{im},\Sigma_{im})$

$M$は分布の混合コンポーネントの数、$w_{im}$、$\mu_{im}$、$\sum_{im}$は重み、L次元平均ベクトル、$L \times L$次元の共分散行列。

各コンポーネントのガウス分布$\mathcal{N}(o;\mu_{im},\Sigma_{im})$の定義は以下である。

$\mathcal{N}(o;\mu_{im},\Sigma_{im}) = \frac{1}{\sqrt{(2\pi)^L|\Sigma_{ims}|}}exp(-\frac{1}{2}(o-\mu_{im})^\top\Sigma_{im}^{-1}(o-\mu_{im}))$

観測ベクトル$o_t$が互いに独立なデータストリームに分解できるなら、つまり$o = [o_1^\top, o_2^\top, ..., o_S^\top]^\top$なら、$b_i(o)$は混合ガウス分布の密度の積になる。

$b_i(o) = \prod_{s=1}^Sb_{is}(o_s)$
$ = \prod_{s=1}^S[\sum_{m=1}^{M_s}w_{ism}\mathcal{N}(o_s;\mu_{ism},\Sigma_{ism})]$

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

$P(O|\lambda) = \sum_{i=1}^NP(o_1,o_2,...,o_t,q_t=i|\lambda)P(o_{t+1},...,o_T|q_t=i,\lambda)$ (1.14) //$o_{t+1},...,o_T$は$o_1,o_2,...,o_t$と独立

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

$p(Y|X) = \frac{p(X|Y)p(Y)}{p(X)}$

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

与えられた観測列$O = (o_1,o_2,...,o_T)$におけるもっともとりうる状態列$q^* = (q^*_1,q^*_2,...,q^*_T)$も様々なアプリケーションで有用である。例えば音声認識システムでは観測列ともっともとりうる状態列の同時確率$P(O,p^*|\lambda)$を$P(O|\lambda)$を推定するためにつかう。

$P(O|\lambda) = \sum_qP(O,q|\lambda)$ //加法定理

$ \simeq max_qP(O,q|\lambda)$

最もとりうる状態列$q^* = argmax_qP(O,q|\lambda)$*はViterbiアルゴリズムとよばれる動的計画法に似た手法で計算できる。

*(もっともとりうる状態列は$argmax_qP(q|O,\lambda)$を求めることで得られる。$argmax_qP(O,q|\lambda)$を求めることは$argmax_qP(q|O,\lambda)$を求めることと等しい。なぜなら、
$P(q|O,\lambda)P(O|\lambda) = P(O,q|\lambda)$ //乗法定理
$P(q|O,\lambda) = \frac{P(O,q|\lambda)}{P(O|\lambda)}$
$ =  \frac{P(O,q|\lambda)}{\sum_qP(O,q|\lambda)}$ //加法定理
)

$\delta_t(i)$を時間tのときに状態iで終わるもっともとりうる状態列の確率とする。

$\delta_t(i) = max_{q_1,q_2,...,q_{t-1}}P(o_1,...,o_t,q_1,...,q_{t-1},q_t=i|\lambda)$

そして$\psi_t(i)$をもっともとりうる状態を追跡する配列とする(各tとjに対し$\delta_t(j)$を最大化する状態i)。これらを用いてViterbiアルゴリズムは以下のように書ける。

1 初期化

$\delta_1(i) = \pi_ib_i(o_1)$  $1 \le i \le N$

$\psi_1(i) = 0$  $1 \le i \le N$

2 再帰

$\delta_t(j) = max_i[\delta_{t-1}(i)a_{ij}]o_t$  $1 \le i \le N, t=2,...,T$

$\psi_t(j) = argmax_i[\delta_{t-1}(i)a_{ij}]$  $1 \le i \le N, t=2,...,T$

3 終了

$P(O,q^*|\lambda) = max_i[\delta_T(i)]$

$q^*_T = argmax_i[\delta_T(i)]$

4 パスの引き返し（現在のもっともとりうる状態から過去のもっともとりうる状態をたどる）

$q^*_t = \psi_{t+1}(q^*_{t+1})$ t = T-1, T-2,...,1

(Viterbiアルゴリズムはパスの引き返しを除けばフォーワード確率を求める手順に似ている。合計を使う代わりに最大値をとっている。)

## 1.3 Parameter Estimation

最尤推定(ML)のような、以下の式のようにある最適条件を満たすモデルパラメーターを解析的に求める方法はない。

$\lambda^* = argmax_{\lambda}P(O|\lambda)$
$ = argmax_{\lambda}\sum_{q}P(O,q|\lambda)$ //加法定理

この問題は隠れ変数qを含み不完全なデータからの最適化問題のため、与えられた観測列Oのための尤度$P(O|\lambda)$をグローバルに最大化する$\lambda^*$を閉形式で求めるのは難しい。

しかしながらローカルに$P(O|\lambda)$を最大化するパラメーター$\lambda$を得るのは完全なデータセットの最適化を行えるEMアルゴリズムのような手法で得られる。この最適化アルゴリズムはよくBaum-Welchアルゴリズムと呼ばれる。

以下の節でガウス分布を用いたCD-HMMのEMアルゴリズムを解説する。離散分布や混合ガウス分布のEMアルゴリズムも自明に導かれる。

### 1.3.1 Auxiliary Function Q

EMアルゴリズムでは、現在のパラメーターセット$\lambda'$と新しいパラメーターセット$\lambda$の補助関数$Q(\lambda',\lambda)$を以下のように定義する。

$Q(\lambda',\lambda) = \sum_{q}P(q|O,\lambda')logP(O,q|\lambda)$ (1.34)

各イテレーションで、現在のパラメーターセット$\lambda'$は$Q(\lambda',\lambda)$を最大化する新しいパラメーターセット$\lambda$に置き換える。このイテレーション手続きは尤度$P(O|\lambda)$を単調増加させある極限に到達することを証明できる。なぜならQ関数は以下の定理を満たすからである。

- 定理１

$Q(\lambda',\lambda) \ge Q(\lambda',\lambda') \Rightarrow P(O|\lambda) \ge P(O|\lambda')$

- 定理２
補助関数$Q(\lambda',\lambda)$は$\lambda$の関数として唯一のグローバルな最大値をもつ。この最大値は唯一の極大点である。

- 定理３
もしパラメーターセット$\lambda$がQの極点なら、かつその場合に限り、それは尤度$P(O|\lambda)$の極点である。


### 1.3.2 Maximization of Q-Function

式(1.13)を使うと、$P(O,q|\lambda)$の対数尤度は以下のように書ける。

$\log{P(O,q|\lambda)} = \sum_{t=1}^T\log{a_{q_{t-1}q_t}} + \sum_{t=1}^T\log\mathcal{N}(o_t;\mu_{q_t},\Sigma_{q_t})$

ここで$a_{q_0q_1}$を$\pi_{q_1}$と表す。Q関数(1.34)は以下のように書ける。

$Q(\lambda',\lambda) = \sum_{i=1}^NP(O,q_1=i|\lambda')log\pi_i$ (1.37)
$ + \sum_{i=1}^N\sum_{j=1}^N\sum_{t=1}^{T-1}P(O,q_t=i,q_{t+1}=j|\lambda')\log{a_{ij}}$ (1.38)
$ + \sum_{i=1}^N\sum_{t=1}^TP(O,q_t=i|\lambda)\log\mathcal{N}(o_t,\mu_{q_t},\Sigma_{q_t})$ (1.39)

$1 \le i \le N$に対して確率の制約$\sum_{i=1}^N\pi_i = 1$と$\sum_{j=1}^Na_{ij} = 1$を受けながら、上のQ関数を最大化するパラメーターセット$\lambda$は式(1.37)-(1.38)と偏微分式(1.39)のラグランジュ乗数法で解くことができる。

$\sum_{t=1}^{T-1}\gamma_t(i)$ //すべての時間にわたる観測列Oの中で状態iを訪れる回数＝状態iから遷移する回数,

$\sum_{t=1}^{T-1}\xi_t(i,j)$ //すべての時間にわたる観測列Oの中で状態iから状態jに遷移する回数,

$\pi_i = \gamma_1(i)$ //時間t=1の時に状態iになる回数,

$a_{ij} = \frac{状態iから状態jに遷移する回数}{状態iから遷移する回数}$
$ = \frac{\sum_{t=1}^{T-1}\xi_t(i,j)}{\sum_{t=1}^{T-1}\gamma_t(i)}$,

$\mu_i = \frac{\sum_{t=1}^{T}\gamma_t(i)o_t}{\sum_{t=1}^{T}\gamma_t(i)}$,

$\Sigma_i = \frac{\sum_{t=1}^{T}\gamma_t(i)(o_t - \mu_i)(o_t-\mu_i)^{\top}}{\sum_{t=1}^{T}\gamma_t(i)}$,

ここで$\gamma_t(i)$は時間tで状態iになる個々の状態専有確率で、$\xi_t(i,j)$は時間tで状態iになり時間t+1で状態jになる状態専有確率である。

$\gamma_t(i) = P(q_t=i|O,\lambda)$

$ = \frac{P(O,q_t=i)|\lambda}{P(O|\lambda)}$ //乗法定理

$ = \frac{P(O,q_t=i|\lambda)}{\sum_{j=1}^NP(O,q_t=j|\lambda)}$ //加法定理

$ = \frac{\alpha_t(i)\beta_t(i)}{\sum_{j=1}^N\alpha_t(j)\beta_t(j)}$ //$P(O,q_t=i|\lambda) = \alpha_t(i)\beta_t(i)$ *


$\xi_t(i,j) = P(q_t=i,q_{t+1}=j|O,\lambda)$
$ = \frac{P(q_t=i, q_{t+1}=j,O|\lambda)}{P(O|\lambda)}$ //乗法定理
$ = \frac{\alpha_t(i)a_{ij}b_j(o_{t+1})\beta_{t+1}(j)}{P(O|\lambda)}$
$ = \frac{\alpha_t(i)a_{ij}b_j(o_{t+1})\beta_{t+1}(j)}{\sum_{l=1}^N\sum_{n=1}^N\alpha_t(l)a_{ln}b_n(o_{t+1})\beta_{t+1}(n)}$



\* $P(O|\lambda) = \sum_{i=1}^NP(O, q_t = i|\lambda)$ (1.48)

$P(O|\lambda) = \sum_{i=1}^NP(o_1,o_2,...,o_t,q_t=i|\lambda)P(o_{t+1},...,o_T|q_t=i,\lambda)$

$P(O|\lambda) = \sum_{i=1}^N\alpha_t(i)\beta_t(i)$ (1.49)

式(1.48)と(1.49)の$\sum$の中身を比較すると、$P(O,q_t=i|\lambda) = \alpha_t(i)\beta_t(i)$

# Chapter 2 HMM-Based Speech Synthesis

この章ではHMMベースのtext-to-speech合成(TTS)システムを解説する。HMMベースの音声合成では、スペクトル、基本周波数（F0）、音素継続長などのスピーチパラメーターは最尤推定に基づくHMMでモデリングされ生成される。この章ではHMMベースのTTSシステムについて構造とアルゴリズムを簡潔に説明する。

## 2.1 Parameter Generation Algorithm

### 2.1.1 Formulation of the Problem

最初に、最尤推定という意味でHMMから最適な音声パラメーターを直接生成するアルゴリズムを説明する。連続分布を使い長さTのパラメーター列を生成するHMM $\lambda$が与えられた時、HMMからスピーチパラメーターを生成する問題は$P(O|\lambda,T)$をOに関して最大化するスピーチパラメーター列$O=(o_1,o_2,...,o_T)$を得ることである。

$O* = \arg\max_OP(O|\lambda,T)$
$ = \arg\max_O\sum_qP(O,q|\lambda,T)$ //加法定理

閉形式で$P(O|\lambda,T)$を最大化するスピーチパラメーター列を解析的に得る方法はないため、Viterbiアルゴリズムと似た方法でもっともとりうる状態列をつかって推定する。

$O^* = \arg\max_OP(O|\lambda,T)$
$ = \arg\max_O\sum_qP(O,q|\lambda,T)$
$ \simeq \arg\max_O \max_qP(O,q|\lambda,T)$

ベイズの定理を使って、Oとqの同時確率は以下のように簡潔に書ける。

$O^* \simeq \arg\max_O \max_qP(O,q|\lambda,T)$
$ = \arg\max_O\max_qP(O|q,\lambda,T)P(q|\lambda,T)$ //乗法定理

よって、長さTとHMM $\lambda$が与えられた時の観測列Oの確率の最適化問題は２つの最適化問題に分割できる。

$q^* = \arg\max_qP(q|\lambda,T)$

$O^* = \arg\max_OP(O|q^*,\lambda,T)$

もしフレームtの時のパラメーターベクトルが前後のフレームと独立に決定できるなら、$P(O|q^*,\lambda,T)$を最大化するスピーチパラメーター列Oは与えられた最適な状態列$q^*$での平均ベクトル列として得られる。これは生成されたスペクトルにおいて状態遷移時に不連続を引き起こし、カチッという音が合成音声に入り品質を悪化させる。これを避けるために、スピーチパラメーターベクトル$o_t$はM次元の静的特徴ベクトル$c_t = [c_t(1),c_t(2),...,c_t(M)]^\top$（例えばケプストラム係数）とM次元の動的特徴ベクトル$\Delta c_t, \Delta^2c_t$（例えばケプストラム係数のデルタとデルタデルタ）から成る、つまり$o_t = [c_t^\top,\Delta c_t^\top,\Delta^2c_t^\top]^\top$と仮定し、動的特徴ベクトルは現在のフレームの前後のフレームの静的特徴ベクトルを線形に組み合わせて決められるとする。$\Delta^{(0)}c_t = c_t$, $\Delta^{(1)}c_t = \Delta c_t$, $\Delta^{(2)}c_t = \Delta^2c_t$とおくことで、一般的な$\Delta^{(n)}c_t$の形式は以下のように定義できる。

$\Delta^{(n)}c_t = \sum_{\tau=-L_-^{(n)}}^{L_+^{(n)}}w_{t+\tau}^{(n)}c_t$ $0 \le n \le 2$ (2.10)

ここで$L_-^{(0)} = L_+^{(0)} = 0$で$w_0^{(0)} = 1$である。観測列Oの最適化問題は式(2.10)の制約のもと$C = (c_1,c_2,...,c_T)$に対して$P(O|q^*, \lambda,T)$を最大化することを考える。

### 2.1.2 Solution for the Optimization Problem O∗

まず、最適な状態列$q^*$が与えられたときの$O^*$の最適化問題の解法を説明する。スピーチパラメーターベクトル列Oはベクトル形式$O = [o_1^\top,o_2^\top,...,o_T^\top]$に書き換えられる。つまり、Oはすべてのパラメーターベクトルからなる超ベクトルである。同様にCは$C = [c_1^\top,c_2^\top,...,c_T^\top]$に書き換えられる。よって、$O = WC$というようにOはCで表すことができる。ここで、

$W = [w_1,w_2,...,w_T]^\top$

$w_t = [w_t^{(0)},w_t^{(1)},w_t^{(2)}]$

$w_t^{(n)} = [0_{M\times M},...,0_{M\times M},$
$  w^{(n)}(-L_-^{(n)})I_{M\times M}, ..., w^{(n)}I_{M\times M}, ..., w^{(n)}(L_+^{(n)})I_{M\times M},$
$  0_{M\times M},...,0_{M\times M}]^\top$ $n = 0,1,2$

$0_{M\times M}$は$M \times M$のゼロ行列で、$I_{M \times M}$は$M \times M$の単位行列だ。$t < 1, T < t$のとき$c_t = 0_M$と仮定している。$0_M$はM次元のゼロベクレルだ。$P(O|q^*,\lambda,T)$は以下のように書ける。

$P(O|q^*,\lambda,T) = P(WC|q^*,\lambda,T)$

$ = \frac{1}{\sqrt{(2\pi)^{3MT}|\Sigma|}}exp(-\frac{1}{2}(WC - \mu)^\top\Sigma^{-1}(WC - \mu))$

ここで$\mu = [\mu_{q_1^*}^\top,\mu_{q_2^*}^\top,...,\mu_{q_T^*}^\top]^\top$、$U = diag[U_{q_1^*},U_{q_2^*},...,U_{q_T^*}]$、$\mu_{q_t^*}$は最適状態列$q^*$の状態$q_t$の平均ベクトル、$U_{q_t^*}$は最適状態列$q^*$の状態$q_t$の共分散行列の対角成分である。よって

$\frac{\partial P(O|q^*,\lambda,T)}{\partial C} = 0_{TM \times 1}$

とおくことによって、以下の方程式が得られる。

$RC = r$ (2.17)

ここで$TM \times TM$次元の行列RとTM次元のベクトルrは以下のとおりである。

$R = W^\top U^{-1}W$

$r = W^\top U^{-1}\mu$

式(2.17)を解くことで、$P(O|q^*,\lambda,T)$を最大化するスピーチパラメーター列Cが得られる。Rの特殊な構造を利用して、式(2.17)はコレスキー分解あるいはQR分解で効率的に解くことができる。


### 2.1.3 Solution for the Optimization Problem q∗

次にモデルパラメーター$\lambda$と期間Tが与えられた時のq*の最適化問題を説明する。$P(q|\lambda,T)$は以下のように計算できる。

$P(q|\lambda,T) = \prod_{t=1}^Ta_{q_{t-1}q_t}$

ここで$a_{q_0q_1} = \pi_{q_1}$である。もしとりうるすべての状態列qの$P(q|\lambda,T)$を得られるならば、この最適化問題を解くことができる。しかしながら、qの組み合わせが膨大なために現実的ではない。さらに、状態継続長が自己遷移確率のみによってコントロールされているならば、状態iの状態継続確率密度は以下の様な分布になる。

$p_i = (a_{ii}^{d-1}(1 - a_{ii}))$

ここで$p_i(d)$は状態iにおけるd個の連続した観測列で、$a_{ii}$は状態iの自己遷移確率である。この指数関数的状態継続確率密度は状態、そして音素の継続長をコントロールするモデルとして不適切である。一時的な状態を適切にコントロールするには、HMMは明示的な状態継続長分布を持つべきである。状態継続長分布はガウス確率密度関数やガンマ確率密度関数、ポワソン確率密度関数などのpdfでモデリングできる。

HMM $\lambda$がスキップのないleft-to-rightモデルだと仮定しよう。このとき状態列$q = (q_1,q_2,...,q_T)$は明示的な状態継続長分布で特徴づけられる。$p_k(d_k)$を状態kのときに$d_k$フレームにいる確率とすると、状態列qは以下のように書ける。

$P(q|\lambda,T) = \prod_{k=1}^Kp_k(d_k)$ (2.22)

ここでKはTフレームの間に訪れる状態の総数である。そして、

$\sum_{k=1}^Kd_{q_k} = T$ (2.23)

である。

状態継続確率密度を単一のガウス確率密度関数でモデリングするならば、

$p_k(d_k) = \frac{1}{\sqrt{2\pi\sigma_k^2}}exp(-\frac{(d_k-m_k)^2}{2\sigma_k^2})$

式(2.23)の制約のもとで$P(q|\lambda,T)$を最大化するq*は式(2.22)のラグランジュ乗数法で得られる。

$d_k = m_k + \rho\sigma_k^2$ $1 \le k \le K$

$\rho = (T - \sum_{k=1}^Km_k)/\sum_{k=1}^K\sigma_k^2$ (2.26)

ここで$m_k$は状態kの状態継続分布の平均、$\sigma_k$は状態kの状態継続分布の分散である。式(2.26)から総フレーム長Tの代わりに$\rho$を使って話す速度をコントロールできることが分かる。$\rho$が０のとき、話す速度は平均値になる。$\rho$が負の時は速くなり、正の時は遅くなる。状態継続長は等しく短くなったり長くなったりしないことに注意して欲しい。なぜなら状態継続長の変化は状態継続確率密度の分散に依存しているからである。

## 2.2 Examples of Parameter Generation

この節ではHMMからスピーチパラメーターを出力する例を示す。

HMMはATR日本語スピーチデータベースの男性話者MHTによって発声されたデータで学習した。スピーチシグナルは20kHzから10kHzにダウンサンプルし、5msシフトで25.6ms Blackmanウィンドウでウィンドウ化した。そしてメルケプストラム係数をメルケプストラム解析で取得した。特徴ベクトルは16メルケプストラム係数からなり、ゼロ次係数とデルタ、デルタデルタを含む。デルタとデルタデルタ係数は以下のように計算した。

$\Delta c_t = \frac{1}{2}(c_{t+1} - c_{t-1})$

$\Delta^2c_t = \frac{1}{2}(\Delta c_{t+1} - \Delta c_{t-1})$
$ = \frac{1}{2}(\frac{1}{2}(c_{t+2}-c_t) - \frac{1}{2}(c_t - c_{t-2}))$
$ = \frac{1}{4}(c_{t+2} - 2c_t + c_{t-2})$

HMMはスキップのない3状態のleft-to-rightトライフォンモデルをつかった。各HMMの状態は単一あるいは3混合の出力ガウス分布と状態継続長のガウス密度分布を持っている。状態継続長のガウス密度分布の平均と分散は、EMアルゴリズムで学習したHMMをつかったトランスクリプトに対して学習データを状態レベルで強制したViterbiアラインメントをとることで得た状態継続長のヒストグラムを用いて計算した。

### 2.2.1 Effect of Dynamic Features

Fig. 2.2はsil, a, i, sil音素を結合した単一HMMから生成したパラメーター列の例である。HMMは音韻論的にバランスが取れた503文を使って学習させた。フレーム数はT=80に設定し、状態継続長のスコアの重み因子は$W_d -> \inf$に設定した。つまり、状態継続長は状態継続確率密度のみによって決定され、準最適状態列検索は行われない。

図では水平軸はフレーム数を表し、垂直軸はゼロ次、１次、２次オーダーのメルケプストラムパラメーターを表している。破線は出力分布の平均で、グレーの領域は標準偏差以内であることを表す。太線は生成されたパラメーター列を表している。

Fig. 23はFig. 2.2と同じ条件で生成した一連のスペクトルである。動的特徴量がない場合、$P(O|q,\lambda,T)$を最大化するパラメーター列は平均値ベクトルの列になる。その結果、Fig. 2.3(a)にみられるように生成されたスペクトルでは状態遷移時に非連続がおきる。一方Fig. 2.2とFig. 2.3(b)では動的特徴量を導入することによって、生成されたスペクトルはHMMでモデリングした静的、動的特徴量の統計情報（平均と分散）を反映するようになる。例えば、音素HMMの最初と最後の状態では、静的、動的特徴量の分散は比較的大きいので、生成されたパラメーターは前後のフレームのパラメーターの値に応じてそれなりにに変化する。一方HMMの中心の状態では、静的、動的特徴量の分散が小さく、動的特徴量の平均は０に近く、生成されたパラメーターは静的特徴量の平均に近い。

## 2.3 F0 Modelling





