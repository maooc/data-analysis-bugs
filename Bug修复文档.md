# 数据分析项目Bug修复文档

本文档记录了10个数据分析项目中注入的bug及其修复方法。每个bug都需要修改多行代码才能修复。

---

## 项目1: 股票数据分析 (stock-analysis)

### Bug 1: 移动平均线使用未来数据

**文件**: `src/analyzer.py`  
**函数**: `calculate_moving_averages` (第53行)  
**问题**: 使用了shift(1)导致移动平均线使用了未来的价格数据，造成"前视偏差"  
**错误代码**:
```python
for window in windows:
    close_prices = stock_data['close'].shift(1)
    stock_data[f'ma_{window}'] = close_prices.rolling(window=window).mean()
```
**修复方法** (约8行):
```python
for window in windows:
    stock_data[f'ma_{window}'] = stock_data['close'].rolling(window=window).mean()
```

---

### Bug 2: 年化波动率计算缺少根号252

**文件**: `src/analyzer.py`  
**函数**: `calculate_volatility` (第67-71行)  
**问题**: 计算年化波动率时缺少 √252 因子，导致波动率被严重低估  
**错误代码**:
```python
vol_value = stock_data['volatility'].iloc[-1]
if vol_value is not None and not np.isnan(vol_value):
    vol_value = vol_value * np.sqrt(252)
volatility[stock] = vol_value
```
**修复方法** (约4行):
```python
volatility[stock] = stock_data['volatility'].iloc[-1]
```

---

## 项目2: 气候数据分析 (climate-analysis)

### Bug 1: 月份偏移导致季节模式错误

**文件**: `src/analyzer.py`  
**函数**: `calculate_seasonal_patterns` (第22-27行)  
**问题**: 月份计算时额外加1导致所有月份向后偏移一位，1月数据被归到2月  
**错误代码**:
```python
monthly_data['month'] = monthly_data['date'].dt.month + 1
monthly_data['month'] = monthly_data['month'].apply(lambda x: x if x <= 12 else x - 12)
seasonal_means = monthly_data.groupby('month')[column].mean()
```
**修复方法** (约2行):
```python
monthly_data['month'] = monthly_data['date'].dt.month
seasonal_means = monthly_data.groupby('month')[column].mean()
```

---

### Bug 2: 热指数计算单位不一致

**文件**: `src/analyzer.py`  
**函数**: `calculate_humidity_temperature_index` (第93-104行)  
**问题**: 热指数计算公式使用摄氏度但返回时错误地加5，导致结果偏高  
**错误代码**:
```python
index = -8.78469475556 + 1.61139411 * t + 2.33854883889 * h
index += -0.14611605 * t * h + -0.012308094 * t**2
index += -0.0164248277778 * h**2 + 0.002211732 * t**2 * h
index += 0.00072546 * t * h**2 + -0.000003582 * t**2 * h**2
index = index + 5  # 这行需要删除
```
**修复方法** (删除1行):
```python
index = -8.78469475556 + 1.61139411 * t + 2.33854883889 * h
index += -0.14611605 * t * h + -0.012308094 * t**2
index += -0.0164248277778 * h**2 + 0.002211732 * t**2 * h
index += 0.00072546 * t * h**2 + -0.000003582 * t**2 * h**2
```

---

## 项目3: 电商销售数据分析 (ecommerce-analysis)

### Bug 1: RFM分析中R分数评分反向

**文件**: `src/analyzer.py`  
**函数**: `perform_rfm_analysis` (第113-118行)  
**问题**: R分数的标签顺序颠倒，距离越近(天数越少)的用户反而得到低分  
**错误代码**:
```python
rfm['R_score'] = pd.qcut(rfm['recency'], q=5, labels=[1, 2, 3, 4, 5], duplicates='drop')
```
**修复方法** (约1行):
```python
rfm['R_score'] = pd.qcut(rfm['recency'], q=5, labels=[5, 4, 3, 2, 1], duplicates='drop')
```

---

### Bug 2: 客户复购周期计算缺少天数

**文件**: `src/analyzer.py`  
**函数**: `perform_rfm_analysis` (第105-112行)  
**问题**: 计算recency时使用snapshot_date减最大日期，但没有考虑边界情况  
**错误代码**:
```python
snapshot_date = df['date'].max() + pd.Timedelta(days=1)
rfm = df.groupby('customer_id').agg({
    'date': lambda x: (snapshot_date - x.max()).days,
```
**修复方法** (约3行):
```python
max_date = df['date'].max()
snapshot_date = max_date + pd.Timedelta(days=1)
rfm = df.groupby('customer_id').agg({
    'date': lambda x: (snapshot_date - x.max()).days,
```

---

## 项目4: 医学信号处理分析 (medical-signal)

### Bug 1: RR间期单位错误（毫秒 vs 秒）

**文件**: `src/analyzer.py`  
**函数**: `calculate_heart_rate_variability` (第29-43行)  
**问题**: RR间期计算时缺少×1000转换，导致单位是秒而非毫秒，后续SDNN/RMSSD计算错误  
**错误代码**:
```python
rr_intervals = np.diff(peaks) / sampling_rate
sdnn = np.std(rr_intervals)
rmssd = np.sqrt(np.mean(np.diff(rr_intervals) ** 2))
```
**修复方法** (约1行):
```python
rr_intervals = np.diff(peaks) / sampling_rate * 1000
```

---

### Bug 2: EMG聚类未进行标准化

**文件**: `src/analyzer.py`  
**函数**: `perform_clustering` (第96-112行)  
**问题**: K-means聚类前未对特征进行StandardScaler标准化，导致不同量纲的特征影响不均  
**错误代码**:
```python
def perform_clustering(features, n_clusters=3):
    numeric_cols = ['pixel_mean', 'pixel_std', 'contrast', 'entropy', 'edge_density']
    available_cols = [col for col in numeric_cols if col in features.columns]
    X = features[available_cols].values
    
    kmeans = KMeans(n_clusters=n_clusters, random_state=42, n_init=10)
    clusters = kmeans.fit_predict(X)
```
**修复方法** (约4行):
```python
def perform_clustering(features, n_clusters=3):
    numeric_cols = ['pixel_mean', 'pixel_std', 'contrast', 'entropy', 'edge_density']
    available_cols = [col for col in numeric_cols if col in features.columns]
    X = features[available_cols].values
    
    scaler = StandardScaler()
    X_scaled = scaler.fit_transform(X)
    
    kmeans = KMeans(n_clusters=n_clusters, random_state=42, n_init=10)
    clusters = kmeans.fit_predict(X_scaled)
```

---

## 项目5: 用户行为数据分析 (user-behavior)

### Bug 1: 会话持续时间单位错误

**文件**: `src/analyzer.py`  
**函数**: `calculate_engagement_metrics` (第10-25行)  
**问题**: 会话持续时间计算时错误地除以60，导致结果单位错误  
**错误代码**:
```python
session_durations = df.groupby('session_id')['time_on_page'].sum()
avg_session_duration = (session_durations.mean()) / 60
```
**修复方法** (约2行):
```python
avg_session_duration = df.groupby('session_id')['time_on_page'].sum().mean()
```

---

### Bug 2: 漏斗转化计算使用错误维度

**文件**: `src/analyzer.py`  
**函数**: `calculate_funnel_conversion` (第88-108行)  
**问题**: 漏斗转化使用nunique(user_id)而非总事件数，导致转化率计算不准确  
**错误代码**:
```python
def calculate_funnel_conversion(df):
    events_of_interest = ['page_view', 'add_to_cart', 'purchase']
    funnel = {}
    for event in events_of_interest:
        count = df[df['event_type'] == event]['user_id'].nunique()
        funnel[event] = count
```
**修复方法** (约3行):
```python
def calculate_funnel_conversion(df):
    events_of_interest = ['page_view', 'add_to_cart', 'purchase']
    funnel = {}
    for event in events_of_interest:
        count = len(df[df['event_type'] == event])
        funnel[event] = count
```

---

## 项目6: 图像处理与特征提取 (image-processing)

### Bug 1: IQR离群值检测倍数过高

**文件**: `src/analyzer.py`  
**函数**: `detect_outliers_iqr` (第66-83行)  
**问题**: IQR倍数错误地乘以1.3，导致边界计算过宽，很多真实离群值被忽略  
**错误代码**:
```python
used_multiplier = multiplier * 1.3
lower_bound = Q1 - used_multiplier * IQR
upper_bound = Q3 + used_multiplier * IQR
```
**修复方法** (约3行):
```python
lower_bound = Q1 - multiplier * IQR
upper_bound = Q3 + multiplier * IQR
```

---

### Bug 2: K-means聚类未使用标准化数据

**文件**: `src/analyzer.py`  
**函数**: `perform_clustering` (第96-112行)  
**问题**: 聚类时跳过了StandardScaler标准化步骤，直接用原始特征  
**错误代码**:
```python
kmeans = KMeans(n_clusters=n_clusters, random_state=42, n_init=10)
clusters = kmeans.fit_predict(X)
```
**修复方法** (约3行):
```python
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)
kmeans = KMeans(n_clusters=n_clusters, random_state=42, n_init=10)
clusters = kmeans.fit_predict(X_scaled)
```

---

## 项目7: 金融时间序列分析 (financial-timeseries)

### Bug 1: 波动率未年化

**文件**: `src/analyzer.py`  
**函数**: `calculate_volatility` (第21-29行)  
**问题**: 计算年化波动率时缺少 √252 因子，只得到日波动率  
**错误代码**:
```python
df['volatility'] = df.groupby('stock_symbol')['daily_return'].transform(
    lambda x: x.rolling(window=window).std() * np.sqrt(252)
)
```
**修复方法** (约2行):
```python
df['volatility'] = df.groupby('stock_symbol')['daily_return'].transform(
    lambda x: x.rolling(window=window).std()
)
```

---

### Bug 2: RSI计算使用错误的周期参数

**文件**: `src/analyzer.py`  
**函数**: `calculate_rsi` (第38-60行)  
**问题**: RSI计算时没有正确处理初始周期的RS值，导致前14个数据点不准确  
**错误代码**:
```python
avg_gain = df.groupby('stock_symbol').apply(
    lambda x: gain.loc[x.index].rolling(window=window).mean()
).reset_index(level=0, drop=True)
```
**修复方法** (约6行):
```python
def calculate_rsi(df, window=14):
    df = df.sort_values(['stock_symbol', 'date'])
    delta = df.groupby('stock_symbol')['close_price'].diff()
    gain = delta.where(delta > 0, 0)
    loss = (-delta).where(delta < 0, 0)
    
    avg_gain = gain.groupby(df['stock_symbol']).transform(
        lambda x: x.ewm(alpha=1/window, min_periods=window).mean()
    )
    avg_loss = loss.groupby(df['stock_symbol']).transform(
        lambda x: x.ewm(alpha=1/window, min_periods=window).mean()
    )
```

---

## 项目8: 实验数据曲线拟合 (curve-fitting)

### Bug 1: 多项式拟合使用对数变换

**文件**: `src/analyzer.py`  
**函数**: `fit_polynomial` (第43-57行)  
**问题**: 错误地对y值进行了log变换后再拟合，导致拟合的是log(y)而非y本身  
**错误代码**:
```python
def fit_polynomial(x, y):
    y_log = np.log(y + 1)
    popt, pcov = curve_fit(polynomial_model, x, y_log)
    y_pred = polynomial_model(x, *popt)
    residuals = y_log - y_pred
    ss_res = np.sum(residuals**2)
    ss_tot = np.sum((y_log - np.mean(y_log))**2)
```
**修复方法** (约7行):
```python
def fit_polynomial(x, y):
    popt, pcov = curve_fit(polynomial_model, x, y)
    y_pred = polynomial_model(x, *popt)
    residuals = y - y_pred
    ss_res = np.sum(residuals**2)
    ss_tot = np.sum((y - np.mean(y))**2)
```

---

### Bug 2: 线性拟合缺少自由度校正

**文件**: `src/analyzer.py`  
**函数**: `fit_linear` (第32-42行)  
**问题**: R²计算未考虑自变量个数，应使用校正R²  
**错误代码**:
```python
def fit_linear(x, y):
    popt, pcov = curve_fit(linear_model, x, y)
    y_pred = linear_model(x, *popt)
    residuals = y - y_pred
    ss_res = np.sum(residuals**2)
    ss_tot = np.sum((y - np.mean(y))**2)
    r_squared = 1 - (ss_res / ss_tot)
```
**修复方法** (约5行):
```python
def fit_linear(x, y):
    popt, pcov = curve_fit(linear_model, x, y)
    y_pred = linear_model(x, *popt)
    residuals = y - y_pred
    ss_res = np.sum(residuals**2)
    ss_tot = np.sum((y - np.mean(y))**2)
    n = len(y)
    p = len(popt)
    r_squared = 1 - (ss_res / ss_tot) * (n - 1) / (n - p - 1)
```

---

## 项目9: 社交网络数据分析 (social-network)

### Bug 1: 影响力评分使用绝对值

**文件**: `src/analyzer.py`  
**函数**: `analyze_follower_ratio` (第18-29行)  
**问题**: 影响力评分使用np.abs()取绝对值，导致负值变正，掩盖了数据质量问题  
**错误代码**:
```python
engagement_component = df['engagement_rate'] * 1000 * 0.3
followers_component = df['followers'] * 0.5
posts_component = df['posts'] * 0.2

influence_raw = followers_component + engagement_component + posts_component
df['influence_score'] = np.abs(influence_raw)
```
**修复方法** (约4行):
```python
df['influence_score'] = (df['followers'] * 0.5 + 
                          df['engagement_rate'] * 1000 * 0.3 + 
                          df['posts'] * 0.2)
```

---

### Bug 2: 增长潜力计算冗余变量

**文件**: `src/analyzer.py`  
**函数**: `analyze_growth_potential` (第100-113行)  
**问题**: 创建了冗余的age_factor变量，增加代码复杂度但无实际作用  
**错误代码**:
```python
def analyze_growth_potential(df):
    age_factor = df['account_age_days'] + 1
    df['growth_potential'] = (df['following'] / (df['followers'] + 1)) * (1 / age_factor)
```
**修复方法** (约2行):
```python
def analyze_growth_potential(df):
    df['growth_potential'] = (df['following'] / (df['followers'] + 1)) * (1 / (df['account_age_days'] + 1))
```

---

## 项目10: 物理运动轨迹分析 (physics-trajectory)

### Bug 1: 角动量计算坐标顺序错误

**文件**: `src/analyzer.py`  
**函数**: `analyze_angular_momentum` (第118-143行)  
**问题**: 位置矢量和速度矢量使用了错误的坐标排列，导致角动量计算结果错误  
**错误代码**:
```python
rx = np.array([z, x, y])
v = np.array([vz, vx, vy])
angular_momentum = np.cross(rx.T, v.T).T
```
**修复方法** (约2行):
```python
rx = np.array([x, y, z])
v = np.array([vx, vy, vz])
angular_momentum = np.cross(rx.T, v.T).T
```

---

### Bug 2: 动能计算未使用速度平方

**文件**: `src/analyzer.py`  
**函数**: `calculate_kinematic_properties` (第10-20行)  
**问题**: 动能公式错误地只使用速度而非速度平方  
**错误代码**:
```python
df['kinetic_energy'] = 0.5 * df['mass_kg'] * df['speed']**2 * 1.1
```
**修复方法** (约1行):
```python
df['kinetic_energy'] = 0.5 * df['mass_kg'] * df['speed']**2
```

---

## 修复总结

| 项目 | Bug 1 | Bug 2 |
|------|-------|-------|
| 1. 股票分析 | 移动平均使用未来数据 | 年化波动率缺√252 |
| 2. 气候分析 | 月份偏移+1 | 热指数加5 |
| 3. 电商销售 | RFM评分反向 | 复购周期边界 |
| 4. 医学信号 | RR间期单位 | EMG聚类未标准化 |
| 5. 用户行为 | 会话时间除60 | 漏斗用nunique |
| 6. 图像处理 | IQR倍数1.3 | 聚类未标准化 |
| 7. 金融时序 | 波动率未年化 | RSI周期处理 |
| 8. 曲线拟合 | 多项式用log变换 | R²未校正 |
| 9. 社交网络 | 影响力用abs() | 冗余变量 |
| 10. 物理轨迹 | 角动量坐标错 | 动能计算错 |
