Web3 双链智能监控系统 · 技术深度报告
版本：V10
作者：潇楠 Web3 哨兵
适用平台：Windows 10/11（独立桌面应用）

摘要
本系统是一款面向 EVM 兼容链（Ethereum、BSC、Base、Polygon、Arbitrum）与 Solana 链的实时资产监控与智能分析平台。它通过 WebSocket 长连接与 RPC 轮询混合架构，实现对指定钱包地址的原生币、代币、NFT 的全方位监控，并集成 Telegram、Discord、邮件、语音播报等多渠道推送。系统内置多 AI 提供商（DeepSeek、Moonshot）的智能解读能力，可对每笔交易进行风险评分、资金流向分析与投资建议。此外，系统还提供链上项目追踪（CryptoRank、GeckoTerminal）、新项目发现（RootData 数据集成）、Web3 AI 聊天助手、交易仪表盘、悬浮窗实时通知等高级功能。

本文从架构设计、核心模块、关键技术实现、性能优化、安全性等方面进行全面阐述，展示本系统的技术深度与工程成熟度。

1. 系统总体架构
系统采用 主进程 + 独立子进程 的微内核架构，实现 UI 渲染与监控逻辑的物理隔离，确保任一子进程崩溃不影响主窗口及其他链的监控。

text
┌─────────────────────────────────────────────────────────────┐
│                      主进程 (main.py)                        │
│  - PyWebView 窗口 (HTML/JS 前端)                             │
│  - 系统托盘图标 + 悬浮窗管理                                   │
│  - 子进程生命周期管理 (启动/停止/重启)                         │
│  - 配置文件读写 (JSON) 与热加载                               │
│  - 全局异常捕获与日志轮转                                      │
└───────────────┬─────────────────┬───────────────────────────┘
                │                 │
                ▼                 ▼
    ┌───────────────────┐  ┌───────────────────┐
    │  子进程: Evm.py    │  │  子进程: Sol.py    │
    │  - EVM 链 WSS 订阅 │  │  - Solana WSS 订阅 │
    │  - RPC 余额轮询    │  │  - Helius API 解析 │
    │  - NFTScan 集成    │  │  - DAS 元数据查询  │
    │  - SQLite 持久化   │  │  - 详细日志 JSON   │
    └───────────────────┘  └───────────────────┘
                │                 │
                └────────┬────────┘
                         ▼
              ┌─────────────────────┐
              │  外部服务集成层       │
              │  - AI Providers      │
              │  - Telegram/Discord  │
              │  - CryptoRank API    │
              │  - GeckoTerminal API │
              │  - RootData 数据集成  │
              └─────────────────────┘
设计要点：

单实例互斥：通过 Windows 全局互斥体防止重复运行。

UTF-8 全链路：子进程标准输出重定向至前端日志区，支持中文与 emoji。

代理透明：所有 HTTP/WSS 请求统一走 proxy_config.json 配置，支持国内网络环境。

2. 核心功能模块详解
2.1 多链实时资产监控
2.1.1 EVM 链监控（Ethereum、BSC、Base、Polygon、Arbitrum）
原生币余额监控：采用 RPC eth_getBalance 轮询（间隔 60 秒），通过数据库保存历史余额快照，变化时触发通知。

ERC20 / ERC721 / ERC1155 代币与 NFT 监控：通过 WebSocket 订阅 logs 事件，按 topics 过滤转入/转出。支持三种 Transfer 事件格式：

Transfer(address,address,uint256) – 标准 ERC20/ERC721

TransferSingle(address,address,address,uint256,uint256) – ERC1155 单次转账

TransferBatch(address,address,address,uint256[],uint256[]) – ERC1155 批量转账

代币信息解析：通过 eth_call 读取合约的 symbol() 与 decimals() 方法，并缓存结果（LimitedSizeDict 限制 1000 条）。

NFT 元数据增强：集成 NFTScan API，获取 NFT 名称、图片、地板价等信息，一并推送。

2.1.2 Solana 链监控
交易订阅：使用 logsSubscribe 通过 mentions 过滤监控地址，实时接收交易签名。

交易详情解析：通过 Helius DAS API (/v0/transactions) 获取完整交易数据，解析以下内容：

nativeTransfers – SOL 转账

tokenTransfers – SPL 代币转账（含 uiAmount、mint）

events.nft – NFT 交易事件（销售、转赠）

代币元数据：调用 Helius getAsset 方法获取 SPL 代币的符号、名称、价格信息（price_info）。

价格补全：对未能获取价格的代币，回退至 DexScreener API 查询。

2.2 智能去重与数据持久化
EVM 去重：数据库表 tx_history 设置唯一约束 UNIQUE(tx_hash, log_index, address)，避免同一事件因网络重发而被重复处理。

Solana 去重：内存集合 processed_sigs 存储已处理的签名，限制最大 1000 条（自动淘汰旧记录）。

数据库设计：

tx_history：存储每笔交易的链、地址、类型、资产、金额、美元价值、时间戳。

balance_snapshots：存储地址的最近一次原生币余额，用于轮询变化检测。

详细日志归档：Solana 模块按日期生成 detailed_transactions_YYYY-MM-DD.json 文件，记录完整交易上下文，便于审计与回溯。

2.3 多渠道实时推送
系统通过异步线程池实现非阻塞推送，支持以下渠道：

渠道	协议	容错策略	配置方式
Telegram	HTTP Bot API	重试 3 次，指数退避	TG_BOT_TOKEN + TG_CHAT_ID
Discord	Webhook	重试 3 次，支持 embed 格式	DISCORD_WEBHOOK_URL
Email	SMTP/SSL	重试 3 次，临时清除代理环境变量	EMAIL_CONFIG
语音播报	Pygame	重试 3 次，自动重初始化 mixer	NORMAL_ALERT_SOUND / LARGE_ALERT_SOUND
特色机制：

大额交易特殊标记：当交易美元价值超过阈值（默认 100 USD）时，推送标题添加 🚨 图标，语音播报使用独立音频文件。

来源/去向地址自动识别：根据 msg_type（收到/转出）动态调整显示标签。

2.4 AI 智能解读引擎
系统内置 MultiAIClient 类，支持多 AI 提供商自动轮换与特征开关。

2.4.1 支持的 AI 提供商
DeepSeek：基础模型 deepseek-chat，无联网能力。

Moonshot：支持 $web_search 工具调用，可获取实时新闻与价格。

2.4.2 三大 AI 功能
交易深度解读

输入：链名称、类型、资产、金额、美元价值、发送方、接收方、合约地址、NFT 信息。

输出：JSON 格式，包含 insight（解读）、risk（风险等级）、risk_reason（原因）、suggestion（建议）。

容错：对 LLM 返回的非标准 JSON 进行正则提取与字段修复。

情感化短评

生成一句 20 字以内的幽默/俏皮描述，增加推送可读性。

每日交易日报

定时（每日 9:00）从数据库读取过去 24 小时交易记录，生成投资摘要（总流入/流出、最大笔交易、资产分布变化）。

2.4.3 调用策略
AI 解读在后台异步执行（asyncio.create_task + 15 秒超时），不阻塞主通知推送。

多个提供商按配置顺序轮换，单个失败自动切换。

支持按功能独立开关（例如仅启用交易解读，禁用情感短评）。

2.5 链上数据聚合与项目追踪
2.5.1 CryptoRank 项目列表
通过官方 API https://api.cryptorank.io/v2/currencies 获取最新加密货币项目（按创建时间降序）。

内置多 API Key 轮换机制，单个 Key 达到限频后自动切换。

本地缓存（cryptorank_cache/）存储最近 2000 个项目，支持离线查看与分页检索。

2.5.2 GeckoTerminal 新交易对雷达
调用 https://api.geckoterminal.com/api/v2/networks/{network}/new_pools 获取指定链（Solana、Ethereum、BSC、Base、Arbitrum、Polygon）最近新增的流动性池。

解析池子名称、基础代币/报价代币符号、价格、24h 成交量、流动性、交易笔数、FDV、市值等指标。

支持前端搜索（代币名/地址）与分页展示。

2.5.3 RootData 新项目发现（无爬虫表述）
采用 官方 Open API (https://api.rootdata.com/open/get_item) 结合 公开页面数据解析 的双重策略。

多 API Key 轮换，避免单一密钥额度耗尽。

项目详情包含：一句话简介、官网、推特、GitHub、融资金额、标签、成立时间、生态等。

新项目自动存入本地缓存（frontend/data/rootdata_projects.json）并通过悬浮窗实时推送。

2.6 前端交互与配置管理
2.6.1 主界面（PyWebView）
左右双日志区：分别展示 EVM 与 Solana 的实时输出。

状态指示灯：绿色代表子进程运行中，红色为停止。

控制按钮：启动/停止、语音/AI/RootData 开关、清空日志。

主题切换：深色/浅色模式动态切换。

2.6.2 配置热更新
用户可通过“修改配置”菜单在线编辑以下文件：

evm_config.json – EVM 节点、地址、推送渠道等

sol_config.json – Solana 节点、Helius Key、RPC 等

proxy_config.json – 代理开关与地址

frontend/index.html – 主界面 HTML

frontend/mini.html – 悬浮窗界面

保存后自动重启对应子进程，无需重新打包 EXE。

2.6.3 悬浮窗
独立无边框窗口（380×550），置顶显示，支持拖拽移动。

实时接收资产变动与 RootData 新项目通知，通过 window.addFloatingNotification 动态追加。

通知支持分类（asset / rootdata），可导出为 JSON 文件。

2.6.4 Web3 AI 助手
独立聊天模态框，支持多会话管理（新建、重命名、删除）。

可切换 AI 提供商（DeepSeek / Moonshot）。

新闻中心：一键获取 Web3 / 国际 / AI 科技领域的 RSS 新闻（基于 feedparser）。

价格注入：用户询问代币价格时，自动从 Binance / CoinGecko 获取实时报价并注入上下文。

2.7 交易仪表盘（ECharts）
从本地数据库读取 EVM / Solana 的历史交易数据，展示：

今日汇总（流入/流出/交易笔数）

最近 20 笔交易列表（点击可触发 AI 深度解读）

近 7 天资金流入/流出趋势图（柱状图 + 折线图）

数据通过 get_evm_stats() 与 get_sol_stats() API 实时获取。

3. 关键技术实现与优化
3.1 高可用 WSS 连接管理
节点轮换：每条链可配置多个 WSS 节点（如 ETHEREUM_WSS_NODES_IN / OUT），当前节点异常时自动切换至下一个。

指数退避重连：初始重试延迟 5 秒，每次失败后延迟加倍，上限 60 秒。

代理支持：通过 websockets_proxy 库实现 WSS 代理连接，满足国内网络需求。

3.2 内存与性能优化
有限大小缓存：LimitedSizeDict 限制 token_info_cache（1000）、price_cache（500）、nft_info_cache（200），防止内存无限增长。

异步文件写入队列：Solana 的详细日志通过 asyncio.Queue 串行写入，避免并发写 JSON 导致损坏。

数据库连接池：每次操作使用 with sqlite3.connect 短连接，利用 SQLite 的 WAL 模式提升并发读性能。

3.3 跨平台与兼容性
所有路径使用 os.path.join 动态构造，支持开发环境与 PyInstaller 打包后的 sys._MEIPASS 路径。

音频播放针对 Windows 环境优化，支持中文路径与空格。

命令行参数 --main-exe-dir 传递主程序目录，确保子进程能正确读取外部配置文件。

3.4 安全性设计
单实例互斥：防止用户误启动多个副本导致资源竞争。

激活机制：基于机器码（硬盘序列号+主板 UUID）的 HMAC 签名验证，激活信息加密存储于 %APPDATA%\XiaoNanMonitor\license.dat。

日志脱敏：不记录完整私钥或敏感配置，仅记录错误堆栈。

4. 部署与运维
4.1 打包与分发
使用 PyInstaller 将主程序与子进程打包为单个 EXE，内置 Python 运行时。

外部依赖（evm_config.json、sol_config.json、frontend/ 等）放置于 EXE 同级目录，便于用户修改。

4.2 日志与监控
evm_error.log：每日午夜轮转，仅保留当日日志，避免磁盘膨胀。

sol/detailed_logs/：按日期 JSON 文件，无自动清理，建议用户定期手动归档。

前端日志区：通过 window.addEvmLog / window.addSolLog 实时显示子进程 print 输出。

4.3 故障自愈
子进程异常退出时，主进程不自动重启（避免无限循环），用户可通过 UI 手动重启。

单个 WSS 连接断开后，子进程会无限重连并切换节点，不影响其他链。

5. 技术难点与突破
难点	解决方案
EVM 原生币无法通过 WSS 实时获取	采用 RPC 轮询 + 余额快照对比，平衡实时性与资源消耗
ERC1155 批量转账事件的解析	分别处理 TransferSingle 与 TransferBatch，从 data 字段提取 tokenId 与数量
Solana 代币价格缺失	优先从 Helius price_info 获取，失败则回退 DexScreener
AI 返回非结构化 JSON	正则提取 + 字段级 fallback，确保解读仍能展示
多 AI 提供商特征不一致	设计 features 字典，按功能开关动态选择提供商
国内网络 WSS 连接失败	集成 websockets_proxy，支持 HTTP/SOCKS 代理，并提供前端代理配置界面
长时间运行内存增长	所有缓存使用 LimitedSizeDict，数据库连接使用短连接，避免全局对象累积
6. 未来演进方向
多链扩展：增加对 Avalanche、Fantom、Optimism 等 EVM 链的支持。

更丰富的 AI 能力：接入 GPT-4o、Claude 3.5，实现多模态（链上截图+文字）解读。

云端协同：将部分计算密集任务（如历史数据分析）迁移至云端，降低本地资源占用。

移动端适配：通过 WebSocket 推送或 REST API 提供手机端简化版监控。

7. 总结
本系统是一个工程化程度高、功能全面、技术深度扎实的 Web3 桌面应用。它不仅实现了多链、多资产、多渠道的实时监控，更通过 AI 赋能、链上数据聚合、配置热更新等特性，为用户提供了超越传统区块浏览器的智能体验。系统的架构设计充分考虑了稳定性、可扩展性与易用性，是个人开发者独立完成的高质量商业级作品。

技术关键词：异步 I/O、WebSocket 长连接、多链异构解析、AI 多提供商轮换、缓存策略、进程隔离、热配置、数据持久化、多渠道推送。



邮箱：pingdj@vip.qq.com
网站：https://www.ming.store/
软件技术深度报告：https://www.ming.store/tech-report.html

版本 V10 · 2026年3月 · 潇楠出品# Web3
<img width="1922" height="941" alt="image" src="https://github.com/user-attachments/assets/7fb9b032-5f59-4da3-9df7-e60e32bf8420" />
<img width="1922" height="947" alt="image" src="https://github.com/user-attachments/assets/32e537ad-e7f3-4275-955d-fa019b0e2f7e" />
<img width="1922" height="945" alt="image" src="https://github.com/user-attachments/assets/b88879c6-f6f8-4d23-97c5-f70b28118716" />
<img width="1922" height="950" alt="image" src="https://github.com/user-attachments/assets/c6a1e4c9-a02f-4d0e-98fc-e6aa6135caec" />
<img width="1922" height="942" alt="image" src="https://github.com/user-attachments/assets/1a4fc9e5-7726-4bd9-ade8-2faf40bbfd46" />
<img width="1922" height="951" alt="image" src="https://github.com/user-attachments/assets/cdaa2b8f-a8e4-40bf-9805-4e0f6387646b" />







