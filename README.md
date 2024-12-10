# GMGN 聪明钱地址分析项目设计文档

## 项目概述
本项目旨在对GMGN平台上的聪明钱进行深度分析，识别高效投资策略和潜在的市场机会。

## 系统架构
- 数据获取层：`dao_gmgn.py`
- 存储层：`service_gmgn_storage.py`
- 服务层：`service_gmgn.py`
- 缓存层：`token_market_cap_cache.py`, `token_trades_cache.py`

## 地址筛选规则

### 排除规则
#### 1. 资金规模筛选
##### 1.1 SOL余额过滤
- 规则：SOL余额 < 1
- 目的：排除资金极其微小的地址
- 意义：过滤无实际交易能力的钱包

##### 1.2 利润规模筛选
- 总利润规则：`total_profit_u < 0`
- 7天利润规则：`profit_7d < 15,000 USD`
- 30天利润规则：`profit_30d < 25,000 USD`
- 目的：聚焦有实际盈利能力的地址
- 意义：排除长期亏损或微小收益的地址

#### 2. 交易行为分析
##### 2.1 交易频率过滤
- 规则：30天内交易次数 > 3,000次
- 计算方式：`buy_count_30d + sell_count_30d > 3,000`
- 目的：排除异常高频交易地址
- 意义：识别可能存在机器人交易或刷量行为

##### 2.2 投资回报率分析
- 30天ROI规则：`profit_30d / cost_30d < 0.1`
- 买入/卖出利润率均值规则：两者均 < 0.1
- 目的：评估投资效率
- 意义：筛选出投资策略低效的地址

#### 3. 代币交易多样性
##### 3.1 代币数量过滤
- 规则：30天交易币种 < 10个
- 目的：确保地址有足够的交易多样性
- 意义：排除单一币种或交易范围狭窄的地址

##### 3.2 代币利润异常检测
- 最大单币种利润规则：最大代币利润 < 10,000 USD
- 异常利润分布规则：最大利润 > 第2-10名利润总和
- 目的：识别利润分布不均衡的地址
- 意义：发现可能存在操纵市场的异常交易模式

#### 4. 成本与交易成本
##### 4.1 零成本交易过滤
- 7天成本为0规则
- 30天成本为0规则
- 目的：排除没有实际交易成本的地址
- 意义：过滤可能是测试或模拟交易的钱包

#### 5. 状态管理
##### 5.1 删除状态处理
- 已标记删除的地址直接跳过
- 满足任一放弃规则，将状态设置为 `Delete`
- 目的：高效管理地址生命周期

### 排除规则的设计哲学
1. 多维度评估
2. 平衡精确性和包容性
3. 动态调整阈值
4. 持续优化算法

### 规则应用流程
1. 顺序检查每个规则
2. 一旦满足任一规则，立即标记为放弃
3. 记录详细的放弃原因
4. 定期审核和调整规则

### 割跟单识别规则

#### 1. 预筛选条件
- 仅分析非删除状态的地址
- 必须具备 `address_activities` 交易活动记录
- 过滤掉市值 >= 100,000 USD 的代币

#### 2. 时间窗口规则（快速交易检测）
##### 2.1 时间间隔限制
- 检测时间窗口：60秒内
- 目标：识别短时间内的异常交易行为

##### 2.2 交易序列要求
- 必须是连续的买入-卖出交易
- 买入交易必须在卖出交易之前
- 交易时间间隔 <= 60秒

#### 3. 交易数据一致性分析
##### 3.1 数量一致性
- 比较买入和卖出交易的基础数量
- 允许的误差范围：±5%
- 计算公式：`|卖出数量 - 买入数量| / 买入数量 <= 0.05`

##### 3.2 数据验证标准
- 买入金额必须为正
- 卖出金额必须为正
- 排除异常或无效交易记录

#### 4. 割跟单特征识别
##### 4.1 高风险特征
- 60秒内完成买入和卖出
- 交易数量高度一致
- 交易代币市值较低（< 100,000 USD）

##### 4.2 风险标记
- 识别到割跟单特征的地址
- 自动标记为 `Delete` 状态
- 在备注中添加 "割跟单" 标签

#### 5. 日志与监控
- 记录每次检测的详细信息
- 输出可疑地址、代币地址和市值
- 使用 `logger` 记录关键步骤和异常

#### 6. 缓存策略
- 使用 `TokenTradesCache` 缓存交易记录
- 使用 `TokenMarketCapCache` 缓存市值信息
- 减少重复网络请求，提高检测效率

#### 7. 异常处理
- 捕获并记录检测过程中的异常
- 确保单个地址检测失败不影响整体流程

#### 8. 检测输出
- 返回 `potential_harvest_addresses` 列表
- 包含可疑地址、代币地址和市值信息

### 割跟单识别的意义
- 保护投资者利益
- 识别市场中的异常交易行为
- 提供区块链交易透明度
- 帮助投资者规避高风险交易

## 打分规则（总分7.5分）

### 评分维度详解

#### 1. 30天胜率 (1分)
- 直接使用 `win_rate_30d` 作为得分
- 胜率越高，得分越高
- 反映交易成功率

#### 2. 7天盈利 (1分)
- 0-5万美元线性得分
- 计算公式：`min(1.0, max(0.0, profit_7d / 50000))`
- 鼓励短期稳定盈利

#### 3. 30天盈利 (1分)
- 0-10万美元线性得分
- 计算公式：`min(1.0, max(0.0, profit_30d / 100000))`
- 评估中期投资表现

#### 4. 利润稳定性 (0.5分)
- 比较7天和30天盈利比例
  - 比例 >= 2：满分0.5分
  - 1.5 <= 比例 < 2：0.25分
  - 比例 < 1：扣1分
- 反映盈利的持续性和一致性

#### 5. 买入利润均值 (0.3分)
- 评估每次买入的平均盈利情况
- 根据平均买入成本和利润计算

#### 6. 买入利润率均值 (0.7分)
- 评估买入交易的盈利率
- 关注相对收益

#### 7. 卖出利润均值 (0.3分)
- 评估每次卖出的平均盈利情况
- 根据平均卖出成本和利润计算

#### 8. 卖出利润率均值 (0.7分)
- 评估卖出交易的盈利率
- 关注相对收益

#### 9. 买卖风格 (1分)
- 比较买入和卖出利润
- 如果卖出利润/买入利润 > 1.5，给予额外加分
- 奖励高效的交易策略

#### 10. 总盈利 (1分)
- 1-5万美元线性得分
- 计算公式：`min(1.0, max(0.0, (total_profit_u - 10000) / (50000 - 10000)))`
- 评估整体盈利能力

#### 11-16. 盈亏TOP占比 (各0.166分，共1分)
##### TOP盈利占比
- TOP1盈利占比 (0-50%)
- TOP5盈利占比 (0-75%)
- TOP10盈利占比 (0-100%)

##### TOP亏损占比
- 亏损TOP1占比 (0-50%)
- 亏损TOP5占比 (0-75%)
- 亏损TOP10占比 (0-100%)

### 评分标准
- 满分：7.5分
- 优秀：> 5分
- 良好：3-5分
- 一般：1-3分
- 较差：< 1分

### 评分目的
- 全面评估地址的投资能力
- 识别高效交易策略
- 量化投资风险和收益

## 缓存策略
1. 市值缓存 (`TokenMarketCapCache`)
- 缓存有效期：24小时
- 减少重复网络请求
- CSV持久化存储

2. 交易记录缓存 (`TokenTradesCache`)
- 缓存有效期：24小时
- 支持按代币地址和钱包地址缓存
- CSV持久化存储

## 错误处理与重试机制
1. 网络请求重试
- 最大重试次数：3次
- 指数退避策略
- 详细日志记录

## 性能优化
1. 缓存机制
2. 批量处理地址
3. 异常处理与快速失败

## 安全考虑
1. 不存储敏感个人信息
2. 使用安全的网络请求方式
3. 日志脱敏

## 未来改进方向
1. 机器学习风险预测模型
2. 实时监控系统
3. 更复杂的缓存淘汰策略
4. 多维度风险评分算法

## 技术栈
- Python
- Peewee ORM
- MySQL
- loguru
- requests/curl

## 注意事项
- 代码模块化
- 简洁明了的注释
- 类型提示
- 枚举类型增强可读性

## 项目操作SOP

### 环境准备
1. Python版本：3.8+
2. 安装依赖库
```bash
pip install peewee loguru requests pandas
```

### 项目文件结构
- `dao_gmgn.py`：数据获取层
- `service_gmgn.py`：核心服务层
- `service_gmgn_storage.py`：存储服务
- `service_gmgn_ut_*`：单元测试和功能验证脚本

### 单元测试脚本执行顺序

#### 1. 获取利润Top 100地址
```bash
python service_gmgn_ut_1_获取利润top100地址.py
```
- 目的：获取盈利最高的100个地址
- 输出：Top 100地址列表

#### 2. 地址统计信息
```bash
python service_gmgn_ut_2_地址统计信息.py
```
- 目的：统计地址的基本信息
- 输出：地址统计报告

#### 3. 初步筛选地址
```bash
python service_gmgn_ut_3_初步筛选地址.py
```
- 目的：根据初步规则过滤地址
- 输出：筛选后的地址列表

#### 4. 所有币的盈亏
```bash
python service_gmgn_ut_4_所有币的盈亏.py
```
- 目的：分析所有代币的盈亏情况
- 输出：代币盈亏详细信息

#### 5. JSON数据检查
```bash
python service_gmgn_ut_5_检查是不是json.py
```
- 目的：验证JSON数据的有效性
- 输出：JSON数据检查报告

#### 6. 二次筛选地址
```bash
python service_gmgn_ut_6_二次筛选地址.py
```
- 目的：进行更严格的地址筛选
- 输出：二次筛选后的地址列表

#### 7. 地址打分
```bash
python service_gmgn_ut_7_打分.py
```
- 目的：对地址进行综合评分
- 输出：地址评分结果

#### 8. 获取地址操作记录
```bash
python service_gmgn_ut_8_获取地址操作记录.py
```
- 目的：获取地址的详细操作历史
- 输出：地址操作记录

#### 9. 检测割跟单地址
```bash
python service_gmgn_ut_9_检测割跟单地址.py
```
- 目的：识别可能的割跟单地址
- 输出：可疑割跟单地址列表

### 注意事项
1. 确保网络连接正常
2. 检查API密钥和权限
3. 关注日志输出
4. 定期备份数据

### 故障排除
- 网络错误：检查网络连接
- API限流：增加重试机制
- 数据解析错误：检查数据格式

### 性能优化建议
1. 使用缓存机制
2. 并行处理数据
3. 优化数据库查询
4. 定期清理缓存
