# another-world-line
1. AI 交互日志
模块	关键 Prompt	AI 输出摘要	决策	AI 犯错案例（错在哪 / 如何发现 / 如何修正）
M1 数据处理	“用pandas读取NYC出租车parquet，清洗异常值，提取小时/星期特征，计算2个衍生特征。”	生成基础过滤、pd.to_datetime转换、trip_duration与avg_speed计算代码。	采用+修改	错在哪：未处理时长为0或负值的情况，直接相除产生inf与NaN。<br>如何发现：运行df['avg_speed'].describe()显示max=inf，且后续建模报错。<br>如何修正：添加.clip(lower=0.1)防除零，补充业务阈值过滤注释。
M2 可视化	“绘制分小时需求折线图，区分工作日/周末，保存为高清png。”	使用groupby+unstack聚合，调用plt.plot()直接出图。	采用	错在哪：图例硬编码为0和1，无坐标轴单位，导出分辨率过低。<br>如何发现：图表放入报告后不符合学术规范，助教评分标准明确标注。<br>如何修正：重命名列名为['Weekday','Weekend']，补充xlabel/ylabel，设置dpi=300与bbox_inches='tight'。
M3 预测模型	“用PyTorch写MLP预测小时订单量，8:2划分，输出MAE/RMSE，与随机森林对比。”	提供完整训练循环、评估指标计算与RF对比代码。	修改后采用	错在哪：输入特征未标准化，导致Loss震荡不收敛；未考虑时间聚合数据的分布偏移。<br>如何发现：训练日志显示Loss在150epoch内未下降，验证集MAE异常偏高。<br>如何修正：引入StandardScaler对特征做归一化，调整学习率至0.01，并在报告中说明聚合数据的非严格时序特性。
M4 问答接口	“实现命令行问答，正则匹配5种问题类型，返回结论+图表路径。”	使用长串if-elif+严格正则（如必须含“请问”“多少”等词）进行路由。	拒绝+重写	错在哪：正则过于死板，用户输入“今天车多吗”“哪个区最堵”均无法命中。<br>如何发现：交互测试10次仅命中3次，用户体验极差。<br>如何修正：弃用严格正则，改用关键词池匹配（any(kw in query.lower() for kw in pool)），增加默认兜底回复，提升容错率。
2. 三阶段对比：以 M1 衍生特征计算 为例
🔹 Native 版（独立编写）
# 手动遍历，逻辑清晰但冗余
durations = []
speeds = []
for _, row in df.iterrows():
    dt = (row['tpep_dropoff_datetime'] - row['tpep_pickup_datetime']).total_seconds() / 60.0
    if dt <= 0: dt = 0.1
    durations.append(dt)
    speeds.append(row['trip_distance'] / dt)
df['duration_min'] = durations
df['avg_speed'] = speeds
效率：⬇️ 低（300万行循环耗时 >40秒）
理解深度：✅ 高（清楚每步计算逻辑，但代码笨重，未利用Pandas向量化优势）
🔹 Prompt 版（AI辅助单次生成）
# AI一键生成，简洁但未考虑边界
df['duration_min'] = (pd.to_datetime(df['tpep_dropoff_datetime']) - df['tpep_pickup_datetime']).dt.total_seconds() / 60.0
df['avg_speed'] = df['trip_distance'] / df['duration_min']
效率：⬆️ 极高（向量化运行 <2秒）
理解深度：⚠️ 中（语法正确，但忽略除零风险与业务合理性，易引发后续建模崩溃）
🔹 Vibe 版（对话驱动迭代）
# 经多轮对话打磨：处理异常值+业务裁剪+注释完善
df['duration_min'] = (pd.to_datetime(df['tpep_dropoff_datetime']) - df['tpep_pickup_datetime']).dt.total_seconds() / 60.0
df['avg_speed'] = (df['trip_distance'] / df['duration_min'].clip(lower=0.1)).clip(0, 120)  # 防除零 & 剔除超速异常
效率：🚀 最高（5行代码，运行 <1秒，直接可用）
理解深度：💡 最深（从“能跑通”升级为“符合交通数据常识”，掌握数据清洗与特征工程的 Trade-off）
📊 对比分析
维度	Native	Prompt	Vibe
开发耗时	长（需查阅API、调试循环）	短（1次生成即得骨架）	中（需多轮引导与验证）
代码质量	冗长但可控	简洁但脆弱	精炼且健壮
认知提升	夯实底层逻辑	突破语法瓶颈	培养工程思维与业务对齐能力
结论：Prompt 版是“加速器”，但容易掩盖数据缺陷；Vibe 版通过持续对话将 AI 从“代码生成器”转化为“结对编程伙伴”，在效率与可解释性之间取得最佳平衡。

3. 反思：对 AI 工具能力边界的新认识
完成本次作业后，我深刻认识到 AI 编程工具的能力边界：它擅长语法生成、范式迁移与重复性代码编写，能极大提升开发效率；但在面对真实数据的噪声处理、业务逻辑的合理性判断以及复杂系统的架构设计时，仍高度依赖人类的领域知识。AI 无法替代对数据字典的深入研读、对异常值的业务溯源，以及对模型过拟合/欠拟合的本质理解。未来的编程不再是“逐行敲击”，而是“精准提问+严格验证+架构把控”。AI 是强大的协作者，但“方向盘”必须掌握在开发者手中，保持批判性思维与工程底线，才能避免陷入“一键生成、一键报错”的陷阱。
