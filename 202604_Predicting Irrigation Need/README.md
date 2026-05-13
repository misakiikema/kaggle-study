# 2026-04 Predicting Irrigation Need — 最終版（vFinal）

## 目的と要約

Kaggleの「Predicting Irrigation Need」コンペにおいて、testデータの各idについて灌漑必要性（Irrigation_Need）を Low / Medium / High の3クラスで予測するモデルを構築した。  
タスクは多クラス分類であり、評価指標は Balanced Accuracy。  
土壌状態・気象条件・作物情報などから、灌漑必要性を分類する。

---

## コンペ概要

課題：灌漑必要性（Irrigation_Need）の予測  
目的変数：Irrigation_Need（Low / Medium / High）  
評価指標：Balanced Accuracy  
提出形式：id,Irrigation_Need  

※ Irrigation_Need 列には Low / Medium / High の予測クラスを提出  
※ 本データは実データではなく合成データ（synthetic data）

---

## 最終アプローチ（概要）

欠損が存在しないため最小限の前処理のみ実施し、LightGBMをベースモデルとして学習。  
StratifiedKFoldによる5-fold CVで評価し、class_weightと正則化によって少数クラス対応と汎化性能の安定化を行った。

---

## 結果

CV Balanced Accuracy = **0.97768**

Public Leaderboard  
LB = **0.96472**

---

## 前処理方針

欠損値は存在しないため補完処理は行っていない。  
目的変数 Irrigation_Need は LabelEncoder により数値ラベルへ変換した。  
id列は識別子のため特徴量から除外した。

カテゴリ特徴量は category 型へ変換し、  
LightGBM の categorical_feature として扱った。

train/test比較では大きな分布差は見られなかったため、  
特別な分布補正は実施していない。

数値特徴量間に強い相関は見られなかったため、  
多重共線性への特別な対応は行っていない。

木モデル（LightGBM）を使用するため、  
スケーリング・標準化・正規化は実施していない。

High クラスが少数であるため、  
class_weight によるクラス重み付けを実施した。

---

## 特徴量のポイント

LightGBM特徴量重要度では、以下の特徴量が上位となった。

- Rainfall_mm（降水量）
- Soil_Moisture（土壌水分量）
- Temperature_C（気温）
- Wind_Speed_kmh（風速）
- Previous_Irrigation_mm（前回の灌漑量）
- Humidity（湿度）

特に、水分量・降水量・気温などの環境系数値特徴量が強く寄与しており、  
モデルが灌漑必要性を主に水分・気象条件から判定していることが確認できた。

また、SHAPによる解釈では「作物の生育ステージ（Crop_Growth_Stage）」の影響も大きく、  
作物状態もクラス判定へ重要な役割を持つことが確認できた。

---

## 学習設定

CV：StratifiedKFold (n=5)<br>
モデル：LightGBM<br>
評価指標：Balanced Accuracy<br>

クラス不均衡対応：<br>
　class_weight = "balanced"<br>

正則化：<br>
　num_leaves = 25<br>
　min_child_samples = 50<br>
　reg_lambda = 1.0<br>

---

## 試した改善

### class_weight（クラス重み付け）

少数クラス（High）を重視するため、  
class_weight = "balanced" を適用。

少数クラスを含めた全体的な予測性能が改善した。

---

### Seed平均

複数seedによる学習を試行。  
予測安定性の向上を目的としたが、今回は大きな改善は確認できなかった。

---

### 正則化

木の複雑さを抑える設定を導入。  

- num_leaves削減
- min_child_samples増加
- reg_lambda追加

を行うことで過学習を抑制し、  
CVスコア改善が確認できた。

---

## 妥当性確認

### 混同行列

Low / Medium / High の各クラスにおいて高い分類性能を確認した。  
特に Low / Medium は非常に高精度で分類できており、  
少数クラスである High についても高いRecallを維持していた。

---

### SHAP

SHAP summary より、

- 土壌水分量
- 気温
- 風速
- 降水量
- 作物の生育ステージ

などがクラス判定へ大きく寄与していることが確認できた。

特に、

- 土壌水分量が低い
- 気温が高い
- 風速が高い

場合に High 判定方向へ寄与する傾向が見られた。

---

## 失敗・効果が小さかった施策

- categorical_feature指定 → 大きなCV改善なし
- Seed平均 → 小幅改善のみ

今回のデータでは、  
特徴量自体の分離性が比較的高く、  
モデル構造変更よりも class_weight と正則化の影響が大きかったと考えられる。

---

## 今後の改善案

- LightGBMパラメータ探索
- CatBoost / XGBoostとの比較
- 特徴量エンジニアリング
- 気象特徴量同士の相互作用特徴量作成
- SHAPを用いた特徴量理解の深化
