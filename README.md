# Alpha Predator (阿尔法捕获者) 📈

Alpha Predator 是一个基于 AI 和大语言模型（LLM）的智能股票分析与交易辅助系统。它结合了传统量化指标、技术面分析、资金流向监控以及大模型的深度推理能力，旨在为投资者提供【可执行、可验证、可复盘】的交易决策支持。

仓库地址: https://github.com/AltraDxx/Alpha-Predator.git

## 版本更新 v1.1.0 (最新)

本次更新主要集中在核心数据稳定性与 UI/UX 体验的深度打磨：

- **⚡️ 极速行情源**：大盘数据源迁移至新浪财经极速接口，延迟从 >60秒 降低至 <100毫秒，秒开体验。
- **📊 明确仓位建议**：买入建议卡片现在会明确给出**“建议买入 1000 股 / 20% 仓位”**，不再模糊。
- **🎨 UI 视觉升级**：全面移除 Emoji，采用统一的高级 SVG 矢量图标系统，界面更加专业纯粹。
- **🔄 智能换仓逻辑**：新增资金不足时的智能换仓建议，自动匹配并建议卖出手中表现不佳的持仓。
- **📉 数据完整性**：修复了深证成指与创业板指在部分接口下缺失的问题。

## 核心功能

### 1. 🎯 Alpha Predator 市场扫描
- **多维捕捉**：识别技术形态突破、底部反转等信号。
- **资金监控**：实时监控主力资金与北向资金流向。
- **交易时段优化**：仅在 A 股交易时间自动刷新，捕捉盘中机会。

### 2. 🔍 Deep Dive 个股深度诊疗
- **六维模型**：覆盖行业、技术、资金、事件、情景推演与综合评分。
- **AI 深度推理**：利用 LLM 生成包含明确买入/卖出评级的专业研报。

### 3. 💼 智能持仓管理可视化
- **盈亏分布**：清晰展示持仓市值与盈亏比例。
- **风控建议**：根据风险偏好自动计算建议的调仓比例。

### 4. ⚙️灵活配置
- **多数据源**：支持 Tushare (推荐) 或 AkShare (默认/免费)。
- **多 LLM 支持**：兼容 OpenAI, Google Gemini, 阿里通义千问 (Qwen) 等。

## 快速开始

### 后端安装
```bash
git clone https://github.com/AltraDxx/Alpha-Predator.git
cd Alpha-Predator
# 创建虚拟环境
python -m venv .venv
source .venv/bin/activate
# 安装依赖
pip install -r requirements.txt
# 启动服务
uvicorn src.api.main:app --reload --port 8000
```

### 前端安装
```bash
cd frontend
npm install
npm run dev
```

访问 `http://localhost:5173` 即可开始使用。

## 架构思路

**数据层 (Data Layer)**：
- 聚合 AkShare/Tushare 硬数据。
- TechnicalIndicators 计算技术指标。

**决策层 (Decision Layer)**：
- 将清洗后的数据注入 PromptTemplate。
- 采用 CoT (思维链) 引导 LLM 进行逻辑推理。

## License
MIT License
