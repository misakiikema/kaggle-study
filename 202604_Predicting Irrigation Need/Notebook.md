# Predicting Irrigation Need (Kaggle)

## 1. 目的と評価指標

### 目的

testデータの各 `id` について、目的変数 **Irrigation_Need** を  
**Low / Medium / High** の3クラスで予測する。

### タスク

多クラス分類

### 評価指標

**Balanced Accuracy**

Balanced Accuracy は、各クラス（Low / Medium / High）の正解率を均等に評価する指標。

クラス数に偏りがある場合でも、少数クラスの予測性能が評価へ反映される。

### 提出形式

`id,Irrigation_Need`

Irrigation_Need 列には  
**Low / Medium / High** の予測クラスを入れる。

### 注意

本コンペのデータは、実データではなく  
**合成データ（synthetic data）** である。

---

## 2. データ定義

土壌状態・気象条件・作物情報などを用いて、  
灌漑必要性を予測する。

主な特徴量の例：

- Soil_Moisture（土壌水分量）
- Rainfall_mm（降水量）
- Temperature_C（気温）
- Wind_Speed_kmh（風速）
- Humidity（湿度）
- Previous_Irrigation_mm（前回の灌漑量）
- Crop_Growth_Stage（作物の生育ステージ）
- Soil_Type（土壌の種類）
- Crop_Type（作物の種類）

目的変数：

- Irrigation_Need
  - Low
  - Medium
  - High

---

## 3. データ粒度確認

### 確認結果

| 項目 | 内容 |
|---|---|
| 1行の単位 | `id` が一意 |
| 重複 | 重複行なし |
| 目的変数 | Irrigation_Need は文字列（Low / Medium / High） |
| クラス分布 | Medium が最多、High が少数 |
| 学習・評価 | 行単位でBalanced Accuracy（多クラス分類） |

High クラスが少数であるため、  
class_weight による不均衡対応を改善候補として検討した。

---

## 4. 欠損・型確認

### 確認結果

| 項目 | 内容 |
|---|---|
| 欠損 | train / testとも欠損なし |
| 型 | 数値列11列・カテゴリ列8列 |
| id | 識別子（int） |
| 異常 | 大きな型不整合なし |

カテゴリ列：

- Soil_Type
- Crop_Type
- Season
- Region
- Irrigation_Type
- Water_Source
- Mulching_Used
- Crop_Growth_Stage

大きなデータ品質問題は見られなかった。

---

## 5. データ可視化

以下を確認した。

- 目的変数割合
- 数値特徴量分布
- カテゴリ特徴量割合
- train/test比較
- 相関ヒートマップ
- SHAP

---

### 目的変数の分布から言えること

Medium クラスが最も多く、  
High クラスが少数である。

そのため、単純Accuracyではなく、  
各クラスを均等に評価する Balanced Accuracy を採用している。

また、学習時には class_weight による少数クラス重視を検討した。

---

### 上位特徴量分布から言えること

#### Soil_Moisture（土壌水分量）

土壌中の水分量を表す特徴量。

Low クラスでは高値側、  
High クラスでは低値側に分布する傾向があり、  
灌漑必要性との関係が明確に見られた。

#### Rainfall_mm（降水量）

降水量を表す特徴量。

High クラスでは低雨量側に寄り、  
Low クラスでは高雨量側に分布する傾向が確認できた。

#### Temperature_C（気温）

高温になるほど蒸発量が増え、  
灌漑必要性が高まる可能性がある。

今回のデータでも、  
High クラスで高温側へシフトする傾向が見られた。

#### Wind_Speed_kmh（風速）

風速が高いほど蒸発が進み、  
土壌乾燥につながる可能性がある。

High クラスでは高風速側へ寄る傾向が見られた。

---

### 相関ヒートマップから言えること

全体として、強い相関は多くなく、  
特徴量同士は概ね独立していた。

最も相関が高かった組み合わせでも、

- Soil_Moisture × Rainfall_mm
- corr ≒ 0.04

程度であり、  
強い多重共線性は見られなかった。

そのため、特徴量削減などの特別な対応は行っていない。

※ 相関は線形関係のみを見るため、  
木モデルでは非線形・閾値・組み合わせで効く可能性がある。

---

## 6. 前処理方針

- 欠損はないため欠損補完は行わない
- 目的変数 Irrigation_Need は LabelEncoder により数値変換する
- `id` は識別子のため特徴量から除外する
- カテゴリ特徴量は category 型へ変換する
- LightGBM の categorical_feature としてカテゴリ特徴量を指定する
- train/test比較で大きな分布差は見られなかったため、特別な補正は行わない
- 強い相関は見られなかったため、多重共線性対応は行わない
- 木モデル（LightGBM）を使用するため、スケーリングは行わない
- High クラスが少数であるため、class_weight によるクラス重み付けを行う

---

## 7. モデル確認

### 7-1. ベースライン

目的：  
LightGBMによるベースライン性能を確認する。

モデル：  
LightGBM

検証：  
StratifiedKFold（5分割交差検証）

評価指標：  
Balanced Accuracy

結果：

- fold1 = 0.97691
- fold2 = 0.97764
- fold3 = 0.97721
- fold4 = 0.97703
- fold5 = 0.97684

CV Balanced Accuracy = 0.97713

---

### 7-2. class_weight

目的：

少数クラス（High）を重視して学習し、  
Balanced Accuracy が改善するかを確認する。

設定：

- class_weight = "balanced"

結果：

- fold1 = 0.97723
- fold2 = 0.97789
- fold3 = 0.97748
- fold4 = 0.97732
- fold5 = 0.97710

CV Balanced Accuracy = 0.97740

---

### 7-3. Seed平均

目的：

seedを変更したモデルを平均し、  
予測安定性向上を確認する。

設定：

- seeds = [42, 2024]
- StratifiedKFold 3分割

結果：

平均CV = 0.97752

---

### 7-4. 正則化

目的：

過学習抑制により、  
CV性能改善を確認する。

設定：

- num_leaves = 25
- min_child_samples = 50
- reg_lambda = 1.0
- class_weight = "balanced"

結果：

- fold1 = 0.97740
- fold2 = 0.97819
- fold3 = 0.97775
- fold4 = 0.97762
- fold5 = 0.97742

CV Balanced Accuracy = 0.97768

---

## 8. 妥当性確認

### 8-1. 特徴量重要度

重要度上位：

- Rainfall_mm（降水量）
- Soil_Moisture（土壌水分量）
- Temperature_C（気温）
- Wind_Speed_kmh（風速）
- Previous_Irrigation_mm（前回の灌漑量）

水分量・気象条件が強く寄与しており、  
灌漑必要性との関係として自然な結果となった。

---

### 8-2. 混同行列

Low / Medium / High の各クラスで高精度な分類を確認。

特に：

- Low
- Medium

は非常に高精度で分類できていた。

High クラスは少数だが、  
Recall も高く維持できていた。

---

### 8-3. SHAP

SHAP summary より、

- Soil_Moisture
- Temperature_C
- Wind_Speed_kmh
- Rainfall_mm
- Crop_Growth_Stage

などが大きく寄与していた。

特に：

- 土壌水分量が低い
- 気温が高い
- 風速が高い

場合に、High 判定方向へ寄与する傾向が確認できた。

また、LightGBM重要度では中位だった  
Crop_Growth_Stage が、SHAPでは高影響となっており、

「分岐回数」よりも  
「予測値への影響量」が大きい特徴量であることが確認できた。

---

## 9. 結論

### 9-0. データ確認結果

欠損・重複・大きなtrain/test差は見られず、  
学習・予測を行えるデータ状態であることを確認した。

また、EDAや特徴量分析を通じて、  
灌漑必要性へ影響しやすい特徴量傾向を把握できた。

---

### 9-1. 最終結果

最終採用：

- LightGBM
- StratifiedKFold（5分割交差検証）

工夫：

- categorical_feature指定
- class_weight
- 正則化

最終スコア：

- CV Balanced Accuracy = 0.97768
- LB = 0.96472

---

### 9-2. 妥当性確認

- 水分・気象条件系特徴量が上位となり、灌漑必要性との関係として自然だった
- SHAPでも同様傾向が確認できた
- 混同行列でも全クラス高精度だった
- 特に High クラスRecallも高く、class_weight の効果が確認できた

---

### 9-3. 施策ごとの学び

| 施策 | 学び |
|---|---|
| categorical_feature指定 | 大きな改善なし |
| Seed平均 | 小幅改善 |
| class_weight | 少数クラス性能改善 |
| 正則化 | 最も改善効果が大きかった |

---

### 9-4. 最終設定

- class_weight = "balanced"
- num_leaves = 25
- min_child_samples = 50
- reg_lambda = 1.0
- StratifiedKFold（5分割）
