# 🛡️ Web3 多功能智能监控系统 · 技术深度报告

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.8+](https://img.shields.io/badge/Python-3.8%2B-blue.svg)](https://www.python.org/)

**版本**：V10
**作者**：潇楠 Web3 哨兵
**适用平台**：Windows 10 / 11（独立桌面应用）

---

## 📄 摘要

本系统是一款面向 **EVM 兼容链**（Ethereum、BSC、Base、Polygon、Arbitrum）与 **Solana** 链的实时资产监控与智能分析平台。它通过 **WebSocket 长连接与 RPC 轮询混合架构**，实现对指定钱包地址的原生币、代币、NFT 的全方位监控，并集成 **Telegram、Discord、邮件、语音播报** 等多渠道推送。

系统内置多 AI 提供商（**DeepSeek、Moonshot**）的智能解读能力，可对每笔交易进行风险评分、资金流向分析与投资建议。此外，系统还提供链上项目追踪（CryptoRank、GeckoTerminal）、新项目发现（RootData 数据集成）、Web3 AI 聊天助手、交易仪表盘、悬浮窗实时通知等高级功能。

**V10 重大更新**：系统内建了 **多聚合器去中心化交易（SWAP）引擎**与 **GoPlus 强制安全检测体系**，实现了从“发现代币”到“安全交易”的完整闭环。用户无需离开软件，即可在多个去中心化交易所聚合器之间自动比价并完成交易；交易前系统会强制进行代币合约安全检测，高风险代币将被自动拦截。

本文从**架构设计、核心模块、关键技术实现、性能优化、安全性**等方面进行全面阐述，展示本系统的技术深度与工程成熟度。

---

## 1. 系统总体架构

系统采用 **主进程 + 独立子进程** 的微内核架构，实现 UI 渲染与监控逻辑的物理隔离，确保任一子进程崩溃不影响主窗口及其他链的监控。


主进程 (main.py)    

PyWebView 窗口 (HTML/JS 前端)  

系统托盘图标 + 悬浮窗管理        

子进程生命周期管理 (启动/停止/重启)    

配置文件读写 (JSON) 与热加载        

全局异常捕获与日志轮转                                     
                 
子进程: Evm.py          

EVM 链 WSS 订阅        

RPC 余额轮询         

NFTScan 集成         

SQLite 持久化                   

子进程: Sol.py    

Solana WSS 订阅

Helius API 解析 

DAS 元数据查询

详细日志 JSON


外部服务集成层     

AI Providers      

Telegram/Discord  

CryptoRank API    

GeckoTerminal API 

RootData 数据集成  

0x / ParaSwap / OKX / Swap API（多聚合器 SWAP）

GoPlus 安全检测
              

**设计要点**：
- **单实例互斥**：通过 Windows 全局互斥体防止重复运行。
- **UTF-8 全链路**：子进程标准输出重定向至前端日志区，支持中文与 emoji。
- **代理透明**：所有 HTTP/WSS 请求统一走 `proxy_config.json` 配置，支持国内网络环境。

---

## 2. 核心功能模块

### 🔷 EVM 链监控
- 原生币通过 **RPC 轮询（60s）** 检测余额变化。
- 代币/NFT 通过 **WSS 订阅 `logs`** 实时捕获 Transfer 事件，支持 ERC20 / ERC721 / ERC1155 事件解析。
- 自动调用 RPC 获取代币 `symbol` 与 `decimals`，集成 **NFTScan** 获取 NFT 元数据与地板价。

### ☀️ Solana 链监控
- 通过 `logsSubscribe` 实时捕获交易签名。
- 调用 **Helius DAS API** 解析 `nativeTransfers`、`tokenTransfers`、NFT 事件。
- 支持 SPL 代币符号与美元价格的自动查询（优先 Helius `price_info`，回退 DexScreener）。

### 🧠 AI 智能解读
- **MultiAIClient** 支持 DeepSeek（无联网）与 Moonshot（联网搜索）轮换。
- 对每笔交易自动生成：**交易意图分析、风险评分（低/中/高）、资金流向解读、投资建议**。
- 支持生成幽默俏皮的“一句话点评”以及每日/每周 AI 交易报告。

### 📡 多渠道推送
- 支持 **Telegram Bot、Discord Webhook、Email（SMTP）、本地语音播报（WAV）**。
- 大额交易（阈值可配置）自动标记为“🚨重磅”并使用特殊推送模板。
- 桌面悬浮窗实时滚动显示最新通知，支持通知分类筛选与导出。

### 💱 内建去中心化交易（SWAP）引擎
- **多聚合器自动比价**：系统依次向 **0x、ParaSwap、OKX DEX、Swap API** 及链上 DEX 合约请求实时报价，自动选择最优路径执行交易。
- **跨链支持**：全面兼容 **Ethereum、BSC、Polygon、Arbitrum、Base**（EVM）以及 **Solana**，覆盖市面上绝大多数的主流代币和流动性池。
- **内置钱包连接与余额显示**：通过外部浏览器完成钱包（MetaMask / Phantom）的安全连接与签名，主界面实时显示当前链钱包余额，并支持一键填入最大可交易数量。
- **交易状态实时同步**：签名完成后，软件自动获取交易哈希并在区块浏览器中追踪交易状态，用户无需离开软件即可掌握交易全流程。

### 🛡️ 强制安全检测与拦截
- **交易前自动检测**：在用户确认交易的瞬间，系统会强制调用 **GoPlus API** 对目标代币合约进行全面的安全扫描。
- **风险分级与处置**：
  - **低风险代币**：自动放行，用户无感知，交易体验流畅。
  - **中风险代币**：弹出详细的风险项目列表（如合约未开源、税率过高、流动性未锁定等），由用户自行评估并决定是否继续交易。
  - **高风险代币**：触发**强制拦截机制**，以醒目的红色警告界面告知用户潜在的资金损失风险，并明确引导用户放弃该笔交易。
- **检测故障容错**：当安全检测因网络波动或服务异常而失败时，系统会明确提示用户，并将是否继续交易的最终决定权交还给用户，避免因检测服务异常而阻断用户的正常交易。

### 📊 链上数据聚合
- **CryptoRank**：实时获取新代币项目列表，支持价格、市值、24h 涨跌幅展示。
- **GeckoTerminal**：新交易对雷达，支持多链筛选与 AI 深度解读。
- **RootData**：新融资项目监控，集成官方 API 与公开数据解析。

### ⚙️ 可视化配置
- 前端界面可直接编辑 **EVM / SOL / 代理** 的 JSON 配置文件。
- 保存后自动重启对应子进程，无需手动重启主程序。
- 支持节点轮换优先级、监控地址列表、推送开关等参数的在线调整。

---

## 3. 关键技术实现

### 3.1 高可用 WSS 连接管理
- 每条链可配置**多个 WSS 节点**，当前节点异常时自动切换至下一可用节点。
- 采用**指数退避重连策略**（5s → 60s），避免频繁重连导致 IP 被限制。
- 通过 `websockets-proxy` 库支持 **HTTP 代理**，解决国内网络访问境外节点的连通性问题。

### 3.2 内存与性能优化
- **`LimitedSizeDict`**：基于 `OrderedDict` 实现的有限容量缓存，自动淘汰最早数据（Token 信息 1000 条，价格 500 条，NFT 元数据 200 条）。
- **异步文件写入队列**：Solana 详细日志通过 `asyncio.Queue` 串行写入，避免多协程并发写 JSON 导致数据损坏。
- **SQLite 优化**：采用短连接 + WAL 模式，避免长事务阻塞写入；交易记录表设置复合唯一约束防止重复入库。

### 3.3 去重与持久化
- **EVM**：利用 SQLite 的 `UNIQUE(tx_hash, log_index, address)` 约束从数据库层面杜绝重复交易。
- **Solana**：内存集合 `processed_sigs` 存储已处理签名，并定期清理早期记录以控制内存占用。
- **隐私保护**：所有交易数据仅存储于本地 SQLite 文件，无任何云端上传。

### 3.4 AI 多提供商容错
- 支持 DeepSeek（无联网）和 Moonshot（联网搜索）两种模式，前端可自由切换。
- 根据配置文件中的 `features` 字段自动判断当前提供商是否支持某类解读任务。
- 对 LLM 返回的**非标准 JSON** 进行正则修复与字段兜底提取，确保前端解读区域永不空白。

---

## 4. 技术难点与突破

| 难点 | 解决方案 |
| :--- | :--- |
| EVM 原生币无法通过 WSS 实时监控 | 采用 **RPC 轮询 + 余额快照对比**，60s 间隔检测，确保不遗漏转账。 |
| ERC1155 批量转账事件解析复杂 | 分别处理 `TransferSingle` 与 `TransferBatch` 事件，从 `data` 字段手动提取 `tokenId` 与 `value` 数组。 |
| Solana 代币价格缺失 | 优先使用 Helius 返回的 `price_info`，若无则实时查询 **DexScreener API** 作为兜底。 |
| 国内网络环境下 WSS 连接频繁失败 | 集成 `websockets-proxy` 代理支持，并通过前端代理配置页面实现可视化修改。 |
| 长时间运行后内存持续增长 | 所有缓存结构均使用 `LimitedSizeDict` 限制最大容量；数据库操作采用短连接，避免连接池堆积。 |
| 桌面应用无法直接安装浏览器钱包插件 | 通过外部浏览器 + 本地 HTTP 服务器桥接架构，实现跨进程钱包连接与签名。 |
| 多个聚合器 API 数据结构差异巨大 | 在报价后端实现统一的数据提取层，兼容 0x、ParaSwap、OKX 等不同返回格式。 |

---

## 5. 安全与隐私

- **数据本地化**：所有监控数据（交易记录、余额快照、地址标签）仅存储于本地 SQLite 数据库，**无任何后台上传**。
- **激活机制**：基于机器码（硬盘序列号 + 主板 UUID）的 **HMAC-SHA256** 签名验证，激活信息经 Base64 加密后存储于 `%APPDATA%` 隐藏目录。
- **交易安全**：内建 SWAP 功能在触发每一笔交易前，均强制调用 GoPlus 进行代币合约安全检测，高风险代币将被自动拦截，保障用户资产安全。
- **进程隔离**：监控子进程与主 GUI 进程完全隔离，单一链的崩溃不影响整体系统运行。
- **单实例互斥**：通过 Windows 互斥体防止多开导致配置文件读写冲突。

---

## 6. 部署与运维

- **打包方案**：使用 **PyInstaller** 将 Python 运行时、依赖库、前端资源、配置文件打包为单个 `exe` 文件，用户下载即用。
- **配置文件外置**：`evm_config.json`、`sol_config.json`、`proxy_config.json` 等配置文件存放于 EXE 同级目录，用户可随时修改，保存后自动重启生效。
- **日志管理**：
  - EVM 日志按**天轮转**（`TimedRotatingFileHandler`），旧日志自动删除。
  - Solana 详细交易记录按日期存储为独立 JSON 文件，方便后期数据分析。
- **容灾恢复**：子进程异常退出后，用户可通过 UI 按钮手动重启，主进程不受影响。

---

## 7. 未来演进方向

- **多链扩展**：增加 Aptos、Sui 等 Move 系公链的监控支持。
- **AI 能力升级**：接入 GPT-4o 等多模态模型，实现交易截图智能分析。
- **云端协同**：推出轻量级云端版，降低本地资源占用，支持多设备数据同步。
- **移动端适配**：通过 WebSocket 推送服务，实现手机端实时通知。

---

## 8. 总结

本系统是一个**工程化程度高、功能全面、技术深度扎实**的 Web3 桌面应用。它不仅实现了多链、多资产、多渠道的实时监控，更通过 **AI 赋能、链上数据聚合、配置热更新** 等特性，为用户提供了超越传统区块浏览器的智能体验。

V10 版本新增的 **内建 SWAP 引擎与强制安全检测体系**，将“发现代币”到“安全交易”的全流程整合于一体，为用户提供了从数据监控、项目评估、多聚合器比价到风险拦截的一站式解决方案。这不仅是对传统 DEX 交易体验的重新定义，更标志着桌面级 Web3 工具开始迈向 **智能化、全闭环** 的新阶段。

系统的架构设计充分考虑了**稳定性、可扩展性与易用性**，是个人开发者独立完成的高质量商业级作品。

---

### 🔑 技术关键词

`异步 I/O` · `WebSocket 长连接` · `多链异构解析` · `AI 多提供商轮换` · `缓存策略` · `进程隔离` · `热配置` · `数据持久化` · `多渠道推送` · `多聚合器 SWAP` · `安全检测拦截`

---

### 📬 联系与资源

- 📧 邮箱：[pingdj@vip.qq.com](mailto:pingdj@vip.qq.com)
- 🌐 官方网站：[https://www.ming.store/](https://www.ming.store/)
- 📖 软件技术深度报告：[https://www.ming.store/tech-report.html](https://www.ming.store/tech-report.html)

---

*版本 V10 · 2026年4月 · 潇楠出品*
<img width="1904" height="933" alt="image" src="https://github.com/user-attachments/assets/62e659ee-31e7-4751-823a-bccbd4be0127" />
<img width="1922" height="959" alt="image" src="https://github.com/user-attachments/assets/baf74acf-5a06-4858-8540-c93c4d6ee121" />
<img width="1922" height="941" alt="image" src="https://github.com/user-attachments/assets/7fb9b032-5f59-4da3-9df7-e60e32bf8420" />
<img width="1922" height="947" alt="image" src="https://github.com/user-attachments/assets/32e537ad-e7f3-4275-955d-fa019b0e2f7e" />
<img width="1922" height="945" alt="image" src="https://github.com/user-attachments/assets/b88879c6-f6f8-4d23-97c5-f70b28118716" />
<img width="1922" height="950" alt="image" src="https://github.com/user-attachments/assets/c6a1e4c9-a02f-4d0e-98fc-e6aa6135caec" />
<img width="1922" height="942" alt="image" src="https://github.com/user-attachments/assets/1a4fc9e5-7726-4bd9-ade8-2faf40bbfd46" />
<img width="1922" height="951" alt="image" src="https://github.com/user-attachments/assets/cdaa2b8f-a8e4-40bf-9805-4e0f6387646b" />
