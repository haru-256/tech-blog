---
title: "超大規模システムでLLM推薦を実現する「ReLand」論文解説とB2B実務への応用戦略"
emoji: "🚀"
type: "tech"
topics: ["機械学習", "推薦システム", "LLM", "RecSys"]
published: false
論文URL: "https://dl.acm.org/doi/10.1145/3640457.3688131"
主な著者: "Changxin Tian, Binbin Hu, et al."
主な所属機関: "Ant Group, Zhejiang University"
論文公開日付: "2024-10-08"
tags: ["RecSys24", "LLM", "推薦システム"]
---

本記事では、2024年のRecSysで発表された論文 **ReLand: Retrieval to effortlessly integrate Large language models' insights into industrial recommenders** を解説します。
本論文では、計算コストの課題を乗り越えて、LLMの高度な推論能力を、数億人規模の実際の推薦システムに組み込むための手法を提示しています。
また、記事著者の考察も記載しています。

> **【画像出典について】** 本記事の論文解説の章に掲載している図・表は、特記のない限り論文「ReLand: Integrating Large Language Models' Insights into Industrial Recommenders via a Controllable Reasoning Pool」（Tian et al., RecSys 2024）から引用しています。

## この論文のすごいところ

この論文の注目すべきポイントは以下の3点です。

1. **リアルタイム推論ではなくベクトル検索でコストの課題を解決**: LLM推論を全ユーザーに対してリアルタイムで行うのではなく、ベクトル検索を用いた「他のユーザーの推論結果で代替する」というアプローチでコスト問題を解決しました。

2. **既存手法と同等の精度を維持しつつ、推論コストを「1/80以下」に劇的圧縮**: 最新のLLM活用手法と比較しても推薦精度を維持しつつ、推論にかかる時間を従来の1/80以下に圧縮しています。

3. **2024年1月からAlipayで本番サービング中**: オフラインでの実験に留まらず、実際にAlipayの大規模環境で稼働しており、オンラインA/BテストでCTR+3.19%、CVR+1.08%の向上を達成し、大きなビジネスインパクトを証明しました。

![論文の要約画像](../images/論文解説%20ReLand%20-%20Integrating%20Large%20Language%20Models'%20Insights%20into%20Industrial%20Recommenders%20via%20a%20Controllable%20Reasoning%20Pool/info-card.png)

## なぜこの論文を選んだか

トラフィックの規模に関わらず、一般的なECサイトにおいて「ユーザーに見られるかどうかも分からない推薦枠」に対して、高コストなLLMをリアルタイムで実行し続けることは困難です。
一方で、LLMが持つオープンワールドな知見や推論能力は極めて有用です。
「低遅延・低コスト」という実務要件と「LLMの恩恵」をどう両立させるか。この課題に対するアプローチになると感じたのが、本論文を選んだ理由です。

## 論文解説

ここからは、論文について解説していきます。

### 概要 (Abstract)

既存の協調フィルタリング（CF）や系列推薦手法は、閉じたデータの枠内で学習するため、LLMのようなオープンワールドで人間的な洞察（インサイト）を抽出することができません。
一方で、LLMの高度な文脈理解を利用しようにも、全ユーザーに対して個別に推論を実行するのは計算コスト的に不可能という課題がありました。

この課題に対し、ReLandは少数のseed userにのみLLM推論を行って推論プールを構築し、残りの一般ユーザーにはベクトル検索でその結果を再利用することで、コストを劇的に圧縮しながら高い有効性とスケーラビリティを実証しました。

### 既存研究とこの論文の位置づけ (Introduction / Related Work)

コスト削減と精度向上のトレードオフにおいて、ReLandがどれだけ画期的かを明確にするため、まずは推薦タスクの基本設定と既存手法の課題を整理します。

#### 推薦システムの問題設定 (Problem Formulation)

一般的なディープラーニングベースの推薦システムは、ItemEncoderを用いてアイテムの埋め込み（Embedding）ベクトル $e_i$ を作り、BehaviorEncoderを用いてユーザーの過去の行動履歴からユーザー埋め込みベクトル $e_u$ を作ります。
そして、これら2つのベクトルからユーザーの興味スコア $y_{ui} = f(e_u, e_i)$ を予測します。
本論文のアプローチは、この基本アーキテクチャを壊さず、「ユーザー埋め込み」を生成する過程にのみLLMのインサイト（推論理由）を注入します。

#### 既存手法の課題

既存のLLMを用いた手法には、直接推薦と知識エンコーダの2種類があります。どちらも全ユーザーに対して個別に推論を実行するため、コストが破綻するという課題があります。

![Fig.1 既存のLLM活用手法の比較（直接推薦 vs 知識エンコーダ）（出典：論文）](../images/論文解説%20ReLand%20-%20Integrating%20Large%20Language%20Models'%20Insights%20into%20Industrial%20Recommenders%20via%20a%20Controllable%20Reasoning%20Pool/fig1.png)

##### 既存手法1（直接推薦）

LLMにプロンプトを与えて直接アイテムを推薦させる手法です。

- **精度のジレンマ:** LLMの常識的推論は新規ユーザー（コールドスタート）には強いですが、データが豊富な一般的なシナリオでは、複雑な行動パターンを学習した従来手法に精度で劣ってしまいます。LLMは「オープンワールドの一般常識」は知っているものの、「ヘビーユーザー特有の、極めて局所的で複雑な購買パターン」までは知りません。そのため、そのまま推薦器として使うと精度が落ちてしまうという根本的な弱点があります。
- **コストの壁:** 速度が遅く実運用不可です。論文内では、1億ユーザーに対してLLaMA-7B・A800 GPU環境で推論させた場合、約1.5万時間（15,180時間）もかかると試算されています。

##### 既存手法2（知識エンコーダ）

LLMを用いてユーザーの好みや推薦の根拠（Rationale）をテキストとして出力させ、それを特徴量として既存モデルに組み込む手法です。
ここで、Rationale（推論理由/根拠）とは、LLMが過去の行動履歴から導き出した「なぜそのユーザーがそのアイテムを好むのか」という言語化されたインサイト（例：「このユーザーは低刺激のスキンケアをまとめ買いする傾向があるため」など）を指します。

- リアルタイム推論の速度問題は解決しましたが、全ユーザーへの個別の推論実行が必要でありコストが破綻します。

これらに対し、本論文の提案手法ReLandは、「LLMによる推論結果（Rationale）を類似ユーザーのRationaleで代用する」というアプローチにより、コスト問題を解決しSOTAを達成しました。

### 提案手法 (Methodology)

#### 手法のコンセプト

ReLand（Retrieval to effortlessly integrate Large language models' insights into industrial recommenders）という名の通り、LLMのリアルタイム推論を放棄し、「推論の非同期バッチ処理」と「ユーザー間でのセマンティック表現の共有」を組み合わせることで、コストを劇的に圧縮します。

このアーキテクチャを実現するため、ReLandは 1. シード抽出、2. LLMによるインサイト出しとプール化、3. 検索による割り当て、4. 推薦モデルへの統合 という明確な4つのステップで構成されています。

![Fig.2 ReLandの全体アーキテクチャ（4ステップ構成）（出典：論文）](../images/論文解説%20ReLand%20-%20Integrating%20Large%20Language%20Models'%20Insights%20into%20Industrial%20Recommenders%20via%20a%20Controllable%20Reasoning%20Pool/fig2.png)

#### Step 1: seed userの抽出（適応的サンプリング）

一般ユーザーに推論を再利用（使い回し）するため、大元となる推論プールには「高品質」かつ「多様な好みや行動パターンを網羅した代表的な知識」を蓄積する必要があります。
この条件を満たすseed user（全体の約10%）を抽出するため、Representativeness-aware SamplingとImportance-aware Samplingという2つのSampling手法を組み合わせます。

##### 1. クラスタリングによる多様性の確保 (Representativeness-aware Sampling)

「ユーザーの好みや意図は過去の行動ログに表れる」という仮定のもと、抽出される推薦理由が一部の嗜好に偏らず、多様な行動パターンを網羅できるように以下の手順を踏みます。

- **行動のベクトル化:** 各ユーザー $u$ の過去のアイテム行動シーケンス $x_u = \{i_1, i_2, \dots, i_{|x_u|}\}$ に対し、軽量なエンコーダ（平均プーリング等）を用いてユーザーの行動ベクトル $\boldsymbol{v}_u$ を生成します。
- **K-means法の適用:** 全ユーザーの行動ベクトル集合 $\{\boldsymbol{v}_u\}$ に対してK-meansクラスタリングを行い、$K$ 個のクラスタに分割します。
- **候補集合の抽出（偏りの排除）:** 形成された各クラスタ $C_k$ ($k=1, \dots, K$) ごとに、そのクラスタの規模（所属ユーザー数）に比例した人数のユーザー $N_k$ をサンプリングし、候補となるユーザー集合 $\mathcal{S}_R$ を構築します。

$$\mathcal{S}_R = \bigcup_{k=1}^K \{N_k \text{ users from } C_k\}$$

##### 2. インタラクション数による質の確保 (Importance-aware Sampling)

単純なランダム抽出は行わず、行動履歴（クリックや購買）が豊富なユーザーを優先的に選出します。
一般ユーザーへ使い回すための「高品質で再利用可能な知識（推論）」をLLMから得るためには、プロンプトの入力となる行動履歴が豊富であることが不可欠です。また、ユーザーの好みが明確に表れていることも同様に重要です。

- **抽出確率の定義:** ユーザー $u$ の行動シーケンスの長さ（インタラクション数）を $\vert x_u \vert$ としたとき、履歴が長いユーザーほど選ばれやすくなるよう、長さに比例したサンプリング確率 $p_u$ を算出します。
- **ベルヌーイ分布によるサンプリング:** 算出された確率 $p_u$ に基づくベルヌーイ分布に従ってサンプリングを行います。

$$u \sim \text{Bernoulli}(p_u)$$

- **候補集合の抽出:** このプロセスを経て、LLMに質の高い推論を行わせるのに十分な情報量を持った「重要（Important）なユーザー」の集合 $\mathcal{S}_I$ を構築します。

##### 3. 最終的なseed user集合の構築（2手法の交互適用とマージ）

上記の「多様性重視（$\mathcal{S}_R$）」と「品質重視（$\mathcal{S}_I$）」の2つのサンプリング手法を交互に適用（alternately apply）していきます。
抽出されたユーザー群を和集合としてマージします。

$$\mathcal{S} = \{u \mid u \in \mathcal{S}_R \cup \mathcal{S}_I\}$$

あらかじめ定義した目標のseed user数（例：全体の10%）に到達するまでこのプロセスを反復して、最終的なseed user集合 $\mathcal{S}$ を構築します。
これにより、全体として「多様性」と「高い推論品質」の両方を備えた推論プールを構築します。

#### Step 2: LLMによる推論生成とプール化 (CoTプロンプティング)

従来の推薦システムは閉じたデータで訓練されるため、「オープンワールドの一般常識」や「人間のような論理的推論能力」が欠如しています。
LLMの高度な文脈理解力を借りることで、推薦の精度と説明性（なぜそれをおすすめするのか）の両方を高めます。

##### 段階的プロンプト (Chain-of-Thought, CoT) の設計

LLMにいきなりアイテムを推薦させるのではなく、プロンプトテンプレート $\mathcal{T}$ を用いて推論プロセスをマクロからミクロへと分割し、精度を上げます。

- **Step 0 (産業用オプション):** ノイズの多い実データ（プロモーションテキスト等）を除去し、ユーザーの行動履歴をクリーンな要約フォーマットに整形します。
- **Step 1 (好みの要約):** 過去の行動履歴から「このユーザーはどのような特徴や好みを持っているか」を自然言語で一文で要約させます。
- **Step 2 (具体的な推薦):** Step 1の要約に基づき、「次にどのようなカテゴリやアイテムに興味を持つか」を推論させます。

![Fig.3 段階的プロンプト（CoT）のステップ設計（出典：論文）](../images/論文解説%20ReLand%20-%20Integrating%20Large%20Language%20Models'%20Insights%20into%20Industrial%20Recommenders%20via%20a%20Controllable%20Reasoning%20Pool/fig3.png)

##### 推論プール（LLM Reasoning Pool）の構築

seed user集合 $\mathcal{S}$ の各ユーザー $s$ に対してLLMから推論テキスト $r_s$ を取得し、推論の集合 $\mathcal{R} = \{r_s \mid s \in \mathcal{S}\}$ を作成します。
そして、seed userの行動ベクトル $\boldsymbol{m}_s$ と推論テキスト $r_s$ をペア（Key-Value）にして、オフラインの辞書である推論プール $\mathcal{P}$ を構築・保存します。

$$\mathcal{P} = \{(\boldsymbol{m}_s, r_s) \mid s \in \mathcal{S}\}$$

#### Step 3: 類似ユーザーへの推論の割り当て (Retrieval)

全ユーザー（数億人）に個別でLLM推論を実行するのはコスト的に破綻します。
しかし、「似た行動パターンを持つユーザーは、推薦される理由（Rationale）も共有できる」という仮定を置けば、seed userの推論結果を検索（Retrieval）して使い回すことができ、LLMのコストを劇的に圧縮しつつ全体にインサイトを拡張できます。

##### 行動ベクトルの生成と検索

推論プールの構築時と同じ軽量なエンコーダを用いて、全ユーザー $u \in \mathcal{U}$ の行動履歴を密なベクトル表現 $\boldsymbol{m}_u$ に変換します。
そして、ベクトル検索エンジン（Proxima等）を使用し、一般ユーザーのベクトル $\boldsymbol{m}_u$ に対して、推論プール内のseed userベクトル $\boldsymbol{m}_s$ との内積（コサイン類似度）を最大化する最適なseed user $s^*$ を特定します。

$$s^* = \mathop{\arg\max}_{s \in \mathcal{S}} \frac{\boldsymbol{m}_u^\top \boldsymbol{m}_s}{\|\boldsymbol{m}_u\| \cdot \|\boldsymbol{m}_s\|}$$

##### 推論の代替

特定した最適なseed user $s^*$ が持つLLM推論テキスト $r_{s^*}$ を、その一般ユーザーに対する推論 $r_u$ としてそのまま代替します（$r_u = r_{s^*}$）。

#### Step 4: 既存モデルへの動的統合 (Dynamic Fusion Layer)

一般的な推薦システム（CF等）が持つ「過去の行動パターンから隠れた関係性を捉える能力（協調シグナル）」と、LLMが持つ「オープンワールドの常識に基づく推論能力」は互いに補完し合う関係にあります。
これらをユーザーごとに最適なバランスで足し合わせ、なおかつ既存システムの大規模なアーキテクチャに手を加えることなく組み込みます。

まず、Step 3で得られた推論テキスト $r_u$ をBERTなどのテキストエンコーダに入力し、意味情報を保持した密ベクトルに変換します。
さらに線形射影層（Linear）を通して次元を $\boldsymbol{e}_u$ に揃えることで、推論ベクトル $\boldsymbol{z}_u$ を生成します。
この変換により、テキスト空間の意味表現を既存モデルが扱える埋め込み空間へと橋渡しします。

$$\boldsymbol{z}_u = \text{Linear}(\text{TextEncoder}(r_u))$$

次に、ユーザーIDベースの埋め込み $\boldsymbol{e}_u$ と推論ベクトル $\boldsymbol{z}_u$ を単純に結合するのではなく、学習可能な重み行列 $W$ を用いたAttentionネットワークにより、ユーザーごとに最適な混合比率を動的に計算します。

$$[\alpha_u^{(1)}, \alpha_u^{(2)}] = \text{softmax}([\boldsymbol{e}_u \cdot W, \boldsymbol{z}_u \cdot W])$$

$$\boldsymbol{e}'_u = \alpha_u^{(1)} \boldsymbol{e}_u + \alpha_u^{(2)} \boldsymbol{z}_u$$

このAttentionにより、行動履歴が豊富なヘビーユーザーには $\alpha_u^{(1)}$ が大きくなって IDベースの埋め込みを重視し、履歴が乏しい新規ユーザーには $\alpha_u^{(2)}$ が大きくなって LLMの推論を重視するという、柔軟なパーソナライズが実現されます。

最後に、融合されたユーザー表現 $\boldsymbol{e}'_u$ を既存のRanker（LightGCNやSASRecなど）に入力します。
**アイテム側のエンコーダや損失関数など、既存システムの主要構造には一切手を加えない**ため、既存パイプラインのアーキテクチャを保ちながら極めて低コストで導入できます。
つまり、ReLandは「ユーザー表現の差し替え」だけで、あらゆる既存推薦システムにLLMのインサイトをプラグイン的に注入できる汎用的な手法です。

### Alipayでの実稼働システム構成 (Industrial Deployment)

理論上のアーキテクチャが、実際の巨大なシステム環境（Alipay）でどのように構築・運用され、コスト要件をクリアしているかを整理します。

数億人規模での低遅延レスポンス（ミリ秒単位）を保証するため、LLM推論は完全にオフラインでバッチ処理され、オンライン側には計算済みのベクトルを渡すだけという、既存のオンラインシステムに負荷をかけない構成をとっています。

![Fig.4 Alipayでの実稼働システム構成（出典：論文）](../images/論文解説%20ReLand%20-%20Integrating%20Large%20Language%20Models'%20Insights%20into%20Industrial%20Recommenders%20via%20a%20Controllable%20Reasoning%20Pool/fig4.png)

#### 採用モデルとハイパーパラメータの詳細

- **採用LLMモデル:** ブラックボックスなAPI（ChatGPT等）に依存せず、プライバシー保護とコスト削減の観点から、オープンソースのChatGLM-6Bを社内のドメインデータでファインチューニングして採用しています。
- **seed userの規模:** 1億人超の全ユーザーの中から、わずか約75,000人を適応的サンプリングで抽出しています（過去30日間の行動データをコンテキストとして使用）。
- **使用エンコーダ:** 推論テキスト（Rationale）のベクトル化にはBERTを使用しています。また、行動エンコーダには、検索・クリック・取引などの行動を統合した異種グラフを構築し、GAT (Graph Attention Networks) を用いてユーザー表現を学習させています（月1回更新）。
- **ベクトル検索エンジン:** 高速な類似ユーザー検索を実現するため、アリババ製のベクトル検索エンジンProximaを採用しています。

#### 驚異的な運用コストの低さ（Practical Analysis）

- **週次バッチ（LLM推論）:** 約75,000人のseed userに対するLLM推論は週に1回実行されます。NVIDIA A100 GPUを4基使用し、わずか約3時間で完了します（1億人全員に推論した場合の半年以上という時間と比較すると、劇的なコスト圧縮です）。
- **日次バッチ（ベクトル検索）:** 全ユーザーの行動ベクトル生成（分散処理）に約9時間、Proximaによる全ユーザーへの推論割り当て（Retrieval）に約2時間かかります。これらは毎日日次バッチで更新されています。
- **オンラインサービング:** 生成されたRationale Embedding行列は、単純な付加特徴量としてオンラインRankerにルックアップされるだけなので、オンラインシステムへの負荷は事実上ゼロに等しいです。

### 実験と結果 (Experiments / Results)

著者は、提案手法の有効性を検証するため、以下の5つのリサーチクエスチョン（RQ）を設定し、客観的なデータに基づいて評価を行っています。

- **RQ1 (Effectiveness):** 提案手法は既存のベースモデル（CF・系列推薦）の性能を底上げできるか？また、他のLLM活用手法と比較して精度面で優位性はあるか？
- **RQ2 (Efficiency):** 提案手法は実稼働に耐えうる推論コスト（トークン数、処理時間）に収まっているか？
- **RQ3 (Ablation & Sensitivity):** 提案手法の各モジュール（適応的サンプリング等）は本当に機能しているか？seed userの割合はどう性能に影響するか？
- **RQ4 (Interpretability):** LLMの推論結果（Rationale）は推薦理由の解釈性向上に寄与するか？
- **RQ5 (Online Impact):** 実際の産業システム（Alipay）にデプロイした際、ビジネス指標（CTR/CVR）は向上するか？

#### 実験設定

##### 1. オフライン実験の設定（RQ1, RQ2, RQ3向け）

- **データセット:** スパース（疎）なデータセットとしてAmazon Sports / Beauty、密なデータセットとしてML-1Mを使用しています。インタラクションが5回未満のユーザー/アイテムは除外する前処理を行っています。
- **データ分割:** 協調フィルタリングタスクでは時間順に「80%訓練 / 10%検証 / 10%テスト」、系列推薦タスクでは最大系列長を50とし、Leave-one-out方式（シーケンスの最後をテスト、最後から2番目を検証、残りを訓練）を採用しています。
- **ベースモデル (Backbones):** 多様なアーキテクチャへの汎用性を検証するため、以下の6種類を使用しています。
    - 協調フィルタリング (CF): NeuMF, NGCF, LightGCN（BPR Lossで最適化）。
    - 系列推薦 (Sequential): GRU4Rec, SASRec, DuoRec（Cross-Entropy Lossで最適化）。
- **比較対象となるLLM手法 (Baselines):** 最新のLLM活用手法であるKARおよびSLIM。
- **ReLandの主要ハイパーパラメータ:**
    - サンプリング: seed userの割合は全体の10%。クラスタ数 $K$ は Sports (1,780), Beauty (1,120), ML-1M (302) に設定。
    - 推論とエンコーダ: LLMにはChatGPT-3.5を使用。軽量行動エンコーダにはLightGCN、テキストエンコーダにはBERTを使用。
    - 学習設定: 埋め込み次元 $d=64$、バッチサイズ 1024、Adamオプティマイザを使用し、patience=10のEarly Stoppingを適用。
- **評価指標:** 推薦タスクの標準的な指標である Hit Ratio (HR@10), Normalized Discounted Cumulative Gain (NDCG@10) を使用しています。

##### 2. オンライン実験の設定（RQ4向け）

- **テスト環境と期間:** Alipayの実際のレコメンドシステム環境において、2024年1月14日〜1月18日の5日間にわたるA/Bテストを実施しました。
- **テスト規模とトラフィック割り当て:** 約2,700万ユーザー（$2.7 \times 10^7$ users）を対象とする大規模なテストです。全体のトラフィックのうち、10%をReLand（提案手法）のグループに、別の10%をベースライングループにランダムに割り当てて比較しています。
- **オンラインベースライン:** 実稼働環境の既存モデルであるMaskNet（ユーザー/アイテムプロファイルやユーザーの行動履歴を特徴量として使用するCTR予測モデル）を採用しています。
- **評価指標:** 実際のビジネスインパクトを厳密に測る Click-Through Rate (CTR) および Conversion Rate (CVR)。

#### 実験結果と考察

##### RQ1: 全体的な性能向上 (Overall Performance)

実験では、ID-only (ベースモデルのみ)、$+f_{item}$ (アイテム属性の追加)、ReLand (LLM推論の追加) の3段階で性能を比較しました。
結果として、全ての手法・データセットにおいてReLandが最高性能（NDCG@10, HR@10）を達成しました。
特に、単純にアイテムの属性情報（$f_{item}$）を結合するよりも、LLMが生成した推論理由（Rationale）を結合する方が高い精度が出ることが証明されました。
また、データがスパースなSports/Beautyにおいてより顕著な改善が見られました。

![Table 2 オフライン実験の全体性能比較（RQ1）（出典：論文）](../images/論文解説%20ReLand%20-%20Integrating%20Large%20Language%20Models'%20Insights%20into%20Industrial%20Recommenders%20via%20a%20Controllable%20Reasoning%20Pool/table2.png)

##### RQ2: LLM手法との効率性比較 (Superiority over LLM-enhanced Methods)

最新手法SLIMと同等の推薦精度を維持しつつ、LLMの推論回数を1%未満に劇的に削減することに成功しました。
処理時間の比較では、SportsデータセットにおいてSLIMが34.57時間かかるのに対し、ReLandは0.41時間で完了しています。

（数式的な背景）: SLIMの計算量が $O(\beta \cdot \vert \mathcal{D} \vert)$ （$\beta$はLLMの推論コスト、$\vert \mathcal{D} \vert $ は全インタラクション数）であるのに対し、ReLandは $O(\beta \vert \mathcal{S} \vert + \gamma \vert \mathcal{U} \vert)$ （ $\gamma$ は検索コスト）で済むため、この大きな差が生まれます。

![Table 3 LLM手法との効率性比較（RQ2）（出典：論文）](../images/論文解説%20ReLand%20-%20Integrating%20Large%20Language%20Models'%20Insights%20into%20Industrial%20Recommenders%20via%20a%20Controllable%20Reasoning%20Pool/table3.png)

##### RQ3: 要因分析とハイパラ感度 (Ablation Study & Hyperparameter Sensitivity)

- **Ablation Study:** w/o r.p. (CoTを使わず直接推論させる) や w/o a.s. (適応的サンプリングの代わりにランダムサンプリングする) などの制限バリアントと比較しました。結果、「ランダムサンプリング (w/o a.s.)」を行うと性能が激減することが判明し、高品質なseed userを抽出するStep 1の戦略が極めて重要であることが裏付けられました。

![Fig.5 Ablation Studyの結果（RQ3）（出典：論文）](../images/論文解説%20ReLand%20-%20Integrating%20Large%20Language%20Models'%20Insights%20into%20Industrial%20Recommenders%20via%20a%20Controllable%20Reasoning%20Pool/fig5.png)

- **seed user数の影響:** 抽出割合を0%〜100%まで変化させたところ、10%を超えたあたりで性能がプラトー（限界効用逓減）に達しました。「全員に推論させてもリソースの無駄であり、10%のseed userで十分に全体の好みをカバーできる」という本手法のコアな主張が証明されました。

![Fig.6 seed user割合と推薦性能の関係（RQ3）（出典：論文）](../images/論文解説%20ReLand%20-%20Integrating%20Large%20Language%20Models'%20Insights%20into%20Industrial%20Recommenders%20via%20a%20Controllable%20Reasoning%20Pool/fig6.png)

##### RQ4: 定性分析と解釈性 (Potential Interpretability of Recommendation)

AmazonのBeautyデータセットにおけるあるユーザーの事例が示されました。
この事例では、ベースモデル (SASRec) が「前回購入したのと同じブランド(Sibu)の別商品」という短絡的な推薦をして失敗しました。
これに対し、ReLandは、LLMが履歴から「低刺激 (hypoallergenic)」「まとめ買い (in bulk)」というユーザーの深い意図をRationaleに言語化していました。そのため、履歴に存在しない別ブランド (CoverGirl) の正解アイテムを正確に推薦することができ、高い解釈性とセマンティック理解を示しました。

![Fig.8 定性分析のケーススタディ（RQ4）（出典：論文）](../images/論文解説%20ReLand%20-%20Integrating%20Large%20Language%20Models'%20Insights%20into%20Industrial%20Recommenders%20via%20a%20Controllable%20Reasoning%20Pool/fig8.png)

##### RQ5: オンラインA/Bテストの結果 (Online Experiment)

Alipayの約2,700万ユーザーを対象とした5日間の本番A/Bテストが実施されました。
全体として **CTR +3.19%, CVR +1.08%** という統計的に有意で巨大なビジネスインパクトを達成しています。

- **完全デプロイ後の長期的な効果:** その後、本番環境に完全デプロイされてから追加で14日間の追跡テストが行われ、そこでもCTR +1.53%, CVR +1.02%の向上が確認され、一過性ではない継続的な有効性が証明されました。
- **セグメント別分析の事実:** ユーザーを活動レベル（$L_1$〜$L_4$）で分割したところ、履歴の少ない低頻度ユーザー層 ($L_1$) ではCVR +10.08%と劇的に改善しました。一方で、最も活発な超ヘビーユーザー層 ($L_4$) ではCVR -0.39%と逆に悪化していました。

![Table 4・5 オンラインA/Bテスト結果とユーザーセグメント別分析（出典：論文）](../images/論文解説%20ReLand%20-%20Integrating%20Large%20Language%20Models'%20Insights%20into%20Industrial%20Recommenders%20via%20a%20Controllable%20Reasoning%20Pool/table4+5.png)

## 記事著者の考察と実務への応用

ここからは、本論文を読んだ著者自身の考察です。

### ABテスト結果のL4層のCVR低下について

実験データを見ると、売上を牽引するはずの超ヘビーユーザー（$L_4$層）において、CVRが「-0.39%」と悪化していることが分かります。
この点は論文では触れられていませんでした。

ヘビーユーザーのCVR低下を招いた原因として、以下の3つの仮説が考えられます。

- **パーソナライズの「不可逆な圧縮ロス」 (Retrievalの弊害):** ヘビーユーザーは複雑で多岐にわたる行動ベクトル $\boldsymbol{m}_u$ を持っています。これを単一のseed user $s^*$ の推論結果に丸め込む（1-NN検索する）ことは、高次元な個人のコンテキストを著しく損なわせるノイズとなるためです。

- **CTRとCVRの乖離（Exploration vs. Exploitationの不均衡）:** LLMの得意とする「意外性（Serendipity）」のある推論が、ユーザーの好奇心を刺激してクリック（CTR）は誘発するが、B2B実務において必須となる「明確な購買意図（Intent）」の裏付けがないため、コンバージョンには至りません。

- **ドメインの意図混線:** Alipayのようなスーパーアプリでは「エンタメ目的（動画視聴等）」と「購買目的」が混在しており、LLMの推論内で意図のノイズが発生した可能性が高いです。

上記のことを考えると、ヘビーユーザーに対しては、LLMのインサイトよりも従来の豊富な行動ログを元にした行動ベクトルを重視するハイブリッドなアプローチが有効であると考えられます。

## まとめ

ReLandの推論のプール化と使い回しは、コスト課題を解決しつつ、LLMの高度なインサイトを推薦に組み込むための優れたアプローチです。
しかし、単純に組み込むだけではヘビーユーザーへの悪影響を招く課題もあるため、動的ゲーティングなどを活用して、ユーザーの特性に合わせて柔軟にアプローチを使い分けることが重要です。

## 参考文献・次に読むべき論文

- [ReLand: Integrating Large Language Models' Insights into Industrial Recommenders via a Controllable Reasoning Pool (本論文)](https://dl.acm.org/doi/10.1145/3640457.3688131)
- [KAR: Towards Open-World Recommendation with Knowledge Augmentation from LLMs](https://arxiv.org/abs/2306.10933)
- [SLIM: Can Small Language Models be Good Reasoners for Sequential Recommendation?](https://arxiv.org/abs/2311.07608)
- [MIND: Multi-Interest Network with Dynamic Routing for Recommendation at Tmall](https://arxiv.org/abs/1904.08030)
