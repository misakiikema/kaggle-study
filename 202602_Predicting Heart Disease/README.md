# 2026-01 Heart Disease Prediction — 最終版（vFinal）

## 目的と要約
KaggleのHeart Disease予測コンペにおいて、testデータの各idについて心臓病が「Presence（あり）」となる確率を予測するモデルを構築した。  
タスクは二値分類であり、評価指標はROC AUC。  
臨床指標データ（年齢・最大心拍数・ST低下量など）から心疾患リスクを推定する。

---

## コンペ概要
課題：Heart DiseaseのPresence確率を予測  
目的変数：Heart Disease（Presence / Absence）  
評価指標：ROC AUC  

提出形式  
id,Heart Disease  

※ Heart Disease列にはPresenceの予測確率を提出　

※ 本データは実データではなく合成データ（generated data）

---

## 最終アプローチ（概要）
欠損がないため最小限の前処理のみ実施し、LightGBMをベースモデルとして学習。  
StratifiedKFoldによる5-fold CVで評価し、seed平均と正則化によって汎化性能を安定化させた。

---

## 結果

CV AUC = **0.95542**

Public Leaderboard  
LB = **0.95344**

最終順位  
**1425位**

---

## 前処理方針

欠損値は存在しないため補完処理は行っていない。  
目的変数 Heart Disease は Presence=1 / Absence=0 に変換した。  
id列は識別子のため特徴量から除外した。

特徴量は数値型だが、一部の列はカテゴリ的性質を持つ可能性があるため  
LightGBMのcategorical_featureとして指定する実験を実施した。

木モデル（LightGBM）を使用するためスケーリングは実施していない。

---

## 特徴量のポイント

モデルの重要度上位には以下の特徴量が現れた。

- Thallium（タリウム負荷検査）
- Chest pain type（胸痛タイプ）
- Max HR（最大心拍数）
- Number of vessels fluro（造影で確認された血管数）
- Exercise angina（運動誘発性狭心症）
- ST depression（ST低下量）

これらは心筋虚血や血管狭窄を直接反映する指標であり、医学的にも整合的な結果となった。

---

## 学習設定

CV  
StratifiedKFold (n=5)

モデル  
LightGBM

seed平均  

[42, 2024, 777, 13, 999]

正則化  

num_leaves = 25  
min_child_samples = 50  
reg_lambda = 1.0  

---

## 試した改善

### カテゴリ特徴量指定
LightGBMのcategorical_featureとしてカテゴリ列を指定。  
結果：CVスコアの改善はほぼ見られなかった。

### Seed平均
複数seedで学習したモデルの予測確率を平均。  
小幅だがCVスコアが改善し、予測の安定性が向上した。

### 正則化
木の複雑さを抑える設定を導入。  
CV / LBともに最も改善効果が見られた。

### learning_rate調整
learning_rateを下げて学習を丁寧に行う設定を試したが、  
正則化単独以上の改善は確認できなかった。

---

## 失敗・効果が小さかった施策

- categorical_feature指定 → CV改善なし  
- learning_rate調整 → スコア差はほぼなし  

今回のデータは特徴量の分離が比較的明確なため、  
モデル構造よりも正則化とseed平均の影響が大きかったと考えられる。

---

## 今後の改善案

- LightGBMのパラメータ探索（num_leaves / min_child_samples / reg_lambda）
- LogisticRegressionなど他モデルとの比較
- 特徴量エンジニアリング
- 相互作用特徴量の作成
- SHAPを用いた特徴量理解の深化
