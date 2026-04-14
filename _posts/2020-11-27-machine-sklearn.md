---
title: Sklearn 特征工程实战 —— 基于用户行为数据预测路段拥堵
description: >-
  数据和特征决定了机器学习的上限，而模型和算法只是逼近这个上限而已。
  本文记录在交通拥堵预测项目中，使用 sklearn 处理用户行为数据的特征工程实践。
author: morric
date: 2020-11-27 20:55:00 +0800
categories: [开发记录, MachineLearning]
tags: [python, ai, sklearn]
pin: true
---

在交通拥堵预测项目中，除了路口的实时车流量数据，我们还引入了用户行为数据作为辅助特征——包括导航请求频次、历史出行时长、高峰时段出行偏好等。这类数据维度高、量纲不一、异常值多，特征工程的质量直接影响模型效果。

本文记录在这个场景下使用 sklearn 做特征工程的完整流程，包括数据预处理、特征选择和降维三个阶段。

---

## 数据结构

原始数据每一行对应一个用户在某路段的行为记录，主要字段如下：

```python
# 特征说明
# user_id          用户ID
# route_request    该路段导航请求次数（日均）
# avg_duration     历史平均通行时长（秒）
# peak_ratio       高峰时段出行占比（0~1）
# detour_count     绕行次数（异常值较多）
# road_type        路段类型（主干道/次干道/支路，类别特征）
# weather_code     天气编码（类别特征）
# hour_features    24个时段的出行频次（高维，需降维）

# 目标值
# congestion_level  拥堵等级（0=畅通 1=缓行 2=拥堵）
```

导入数据：

```python
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split

df = pd.read_csv('traffic_user_behavior.csv')

# 数值特征
num_features = ['route_request', 'avg_duration', 'peak_ratio', 'detour_count']
# 类别特征
cat_features = ['road_type', 'weather_code']
# 高维时段特征
hour_features = [f'hour_{i}' for i in range(24)]

X = df[num_features + cat_features + hour_features]
y = df['congestion_level']
```

---

## 一、数据预处理

### 1.1 数值范围差异大 —— 标准化处理

`route_request`（日均 0~500 次）和 `peak_ratio`（0~1）的量纲相差悬殊，直接送入模型会导致大数值特征主导距离计算。使用 `StandardScaler` 做标准化：

```python
from sklearn.preprocessing import StandardScaler

scaler = StandardScaler()
X_num_scaled = scaler.fit_transform(df[num_features])

print("标准化前 avg_duration 均值:", df['avg_duration'].mean())
print("标准化后 avg_duration 均值:", X_num_scaled[:, 1].mean().round(6))  # 接近0
```

标准化将每个特征转换为均值为 0、标准差为 1 的分布，消除量纲影响。

对于需要输出概率的场景（如逻辑回归），也可以使用 `MinMaxScaler` 将特征压缩到 `[0, 1]` 区间：

```python
from sklearn.preprocessing import MinMaxScaler

mm_scaler = MinMaxScaler()
X_num_minmax = mm_scaler.fit_transform(df[num_features])
```

> **实际经验**：`avg_duration`（通行时长）的分布非常不均匀，早晚高峰时段的通行时长是深夜的 3~5 倍，`StandardScaler` 效果比 `MinMaxScaler` 更稳定，不容易被极端值拉偏。

### 1.2 异常值处理 —— 截断 + 标记

`detour_count`（绕行次数）存在明显异常值，个别记录的绕行次数超过 200 次，明显是数据采集问题。直接标准化会让这些异常值严重干扰模型。

策略：先用分位数截断，再标准化，同时新增一列标记该条记录是否曾被截断：

```python
# 计算99分位数作为截断上限
upper = df['detour_count'].quantile(0.99)

# 标记异常值
df['detour_outlier'] = (df['detour_count'] > upper).astype(int)

# 截断
df['detour_count_clipped'] = df['detour_count'].clip(upper=upper)

print(f"截断阈值: {upper}, 异常记录数: {df['detour_outlier'].sum()}")
```

> **实际经验**：保留 `detour_outlier` 这个标记特征很重要。被截断的记录往往对应真实的高频绕行用户，这个信号本身对拥堵预测有正向贡献，不能直接丢掉。

### 1.3 类别特征编码 —— One-Hot 编码

`road_type` 和 `weather_code` 是类别特征，不能直接作为数值输入模型，使用 `OneHotEncoder` 编码：

```python
from sklearn.preprocessing import OneHotEncoder

encoder = OneHotEncoder(sparse=False, handle_unknown='ignore')
X_cat_encoded = encoder.fit_transform(df[cat_features])

# 查看编码后的列名
encoded_cols = encoder.get_feature_names_out(cat_features)
print("编码后类别特征列：", encoded_cols)
# 输出示例：['road_type_主干道' 'road_type_次干道' 'road_type_支路' 'weather_code_晴' ...]
```

> **注意**：`handle_unknown='ignore'` 是必要的，预测时可能出现训练集中没见过的天气编码，忽略而非报错是更安全的处理方式。

### 1.4 合并所有预处理特征

```python
import numpy as np

X_processed = np.hstack([
    X_num_scaled,                          # 标准化后的数值特征
    df[['detour_outlier']].values,         # 异常值标记
    X_cat_encoded,                         # One-Hot 编码的类别特征
    df[hour_features].values               # 时段特征（待降维）
])

print("预处理后特征维度:", X_processed.shape)
# 输出示例：(50000, 4 + 1 + 类别数 + 24)
```

---

## 二、特征选择

预处理后特征维度较高，需要筛选出真正有效的特征。

### 2.1 方差过滤 —— 去除低信息量特征

某些时段（如凌晨 2~4 点）出行数据极少，对应的 `hour_2`、`hour_3` 等特征方差接近 0，对模型没有区分能力：

```python
from sklearn.feature_selection import VarianceThreshold

selector = VarianceThreshold(threshold=0.01)
X_var_filtered = selector.fit_transform(X_processed)

removed = X_processed.shape[1] - X_var_filtered.shape[1]
print(f"方差过滤移除了 {removed} 个低方差特征")
```

### 2.2 基于树模型的特征重要性筛选

使用 GBDT 作为基模型，通过 `SelectFromModel` 选出重要特征：

```python
from sklearn.feature_selection import SelectFromModel
from sklearn.ensemble import GradientBoostingClassifier

gbdt = GradientBoostingClassifier(n_estimators=100, random_state=42)
gbdt.fit(X_var_filtered, y)

# 选择重要性高于均值的特征
selector_model = SelectFromModel(gbdt, prefit=True, threshold='mean')
X_selected = selector_model.transform(X_var_filtered)

print(f"特征选择后维度: {X_selected.shape[1]}")
```

查看各特征的重要性排名：

```python
import pandas as pd

feature_importance = pd.Series(
    gbdt.feature_importances_
).sort_values(ascending=False)

print("Top 10 重要特征：")
print(feature_importance.head(10))
```

> **实际经验**：`peak_ratio`（高峰出行占比）和 `avg_duration`（历史通行时长）始终排在特征重要性的前列，验证了这两个特征与拥堵的强相关性。而部分天气编码特征重要性接近 0，说明在这个数据集中天气对拥堵的直接预测贡献有限。

---

## 三、降维

24 个时段特征之间存在强相关性（如早高峰 7~9 点的出行模式高度相似），使用 PCA 降维既能减少冗余，也能降低过拟合风险。

### 3.1 PCA 主成分分析

```python
from sklearn.decomposition import PCA
import matplotlib.pyplot as plt

# 先单独对时段特征做 PCA
pca = PCA(n_components=0.95)  # 保留95%的方差信息
X_hour_pca = pca.fit_transform(df[hour_features].values)

print(f"原始时段特征维度: 24")
print(f"PCA 降维后维度: {X_hour_pca.shape[1]}")
print(f"各主成分方差贡献率: {pca.explained_variance_ratio_.round(3)}")
```

查看累计方差贡献率，确认降维效果：

```python
cumsum = np.cumsum(pca.explained_variance_ratio_)
print(f"前3个主成分累计解释方差: {cumsum[2]:.1%}")
# 实际结果约为 87%，说明24个时段特征可以压缩到3~5维
```

将 PCA 降维后的时段特征与其他特征合并：

```python
X_final = np.hstack([
    X_selected,     # 经过特征选择的特征
    X_hour_pca      # PCA 降维后的时段特征
])

print(f"最终特征维度: {X_final.shape[1]}")
```

---

## 四、完整 Pipeline

实际工程中，建议用 `Pipeline` 将所有步骤串联，避免训练集和测试集处理不一致的问题：

```python
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.decomposition import PCA
from sklearn.ensemble import GradientBoostingClassifier

# 数值特征处理
num_transformer = Pipeline([
    ('scaler', StandardScaler())
])

# 类别特征处理
cat_transformer = Pipeline([
    ('onehot', OneHotEncoder(handle_unknown='ignore'))
])

# 时段特征处理
hour_transformer = Pipeline([
    ('scaler', StandardScaler()),
    ('pca', PCA(n_components=5))
])

# 组合所有特征处理
preprocessor = ColumnTransformer([
    ('num', num_transformer, num_features),
    ('cat', cat_transformer, cat_features),
    ('hour', hour_transformer, hour_features)
])

# 完整训练流程
pipeline = Pipeline([
    ('preprocessor', preprocessor),
    ('classifier', GradientBoostingClassifier(n_estimators=200, random_state=42))
])

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

pipeline.fit(X_train, y_train)
print(f"测试集准确率: {pipeline.score(X_test, y_test):.4f}")
```

---

## 五、总结

在这个交通拥堵预测项目中，特征工程的几个关键经验：

**异常值不要直接丢掉**。高频绕行用户本身就是拥堵的信号，截断后保留异常标记比直接删除效果更好。

**量纲统一是基础**。用户行为数据中各特征的数值范围差异极大，不做标准化的话，基于距离的模型（如 KNN、SVM）会完全失效。

**时段特征的冗余很高**。24 个时段特征看似信息丰富，实际上早晚高峰之间高度相关，PCA 降到 5 维后模型效果基本没有损失，但训练速度明显提升。

**用 Pipeline 而不是手动拼接**。手动对训练集做 `fit_transform`、对测试集只做 `transform` 很容易出错，Pipeline 强制保证了两者处理流程的一致性，也便于后续的交叉验证和超参数调优。