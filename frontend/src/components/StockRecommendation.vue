<script setup lang="ts">
/**
 * 股票推荐卡片组件
 * 显示推荐股票的详细信息
 */
import { computed, onMounted, ref } from 'vue'

interface PositionStrategy {
  initial_pct: number
  add_condition: string
  max_pct: number
}

interface SellCriteria {
  min_loss_pct?: number
  max_score_threshold?: number
  bad_sectors?: string[]
  reason?: string
}

interface StockRecommendation {
  rank: number
  ts_code: string
  name: string
  sector: string
  signal: string
  score: number
  current_price: number
  buy_price?: number
  sell_price?: number
  target_price?: number  // 兼容旧格式
  stop_loss_price: number
  position_pct?: number  // 兼容旧格式
  position_strategy?: PositionStrategy
  hold_period: string
  entry_timing: string
  reasons: string[]
  risk_factors: string[]
  sell_criteria?: SellCriteria // 新增：智能卖出标准
  replacement_advice?: string
}

const props = defineProps<{
  stock: StockRecommendation
}>()

// 持仓数据
const availableCapital = ref(0)
const positions = ref<any[]>([])

onMounted(() => {
  // 从 localStorage 读取持仓
  const saved = localStorage.getItem('portfolio')
  if (saved) {
    try {
      const portfolio = JSON.parse(saved)
      const totalCapital = portfolio.total_capital || 100000
      const marketValue = (portfolio.positions || []).reduce((sum: number, pos: any) => {
        return sum + pos.quantity * pos.cost_price
      }, 0)
      availableCapital.value = totalCapital - marketValue
      positions.value = portfolio.positions || []
    } catch (e) {
      console.error('读取持仓失败')
    }
  }
})

// 计算建议买入股数或换仓建议
const tradeSuggestion = computed(() => {
  const positionPct = props.stock.position_strategy?.initial_pct || props.stock.position_pct || 20
  const buyPrice = props.stock.buy_price || props.stock.current_price
  
  if (!buyPrice || buyPrice <= 0) return { type: 'hold', message: '价格数据无效', shares: 0 }

  // 1. 资金充足情况：计算可买股数
  const budgetForStock = availableCapital.value * (positionPct / 100)
  const minCost = buyPrice * 100
  
  // 如果可用资金足够买入一手，优先建议买入
  if (budgetForStock >= minCost) {
    const rawShares = Math.floor(budgetForStock / buyPrice / 100) * 100
    const amount = rawShares * buyPrice
    return {
      type: 'buy',
      shares: rawShares,
      amount: amount,
      message: `建议买入 ${rawShares} 股`
    }
  } 
  
  // 2. 资金不足情况：智能匹配换仓
  if (props.stock.sell_criteria && positions.value.length > 0) {
    const criteria = props.stock.sell_criteria
    const candidates = []

    for (const pos of positions.value) {
      // 计算亏损比例
      let lossPct = 0
      if (pos.current_price && pos.cost_price > 0) {
        lossPct = (pos.cost_price - pos.current_price) / pos.cost_price * 100
      }
      
      // 匹配条件 1: 亏损超过阈值
      if (criteria.min_loss_pct && lossPct >= criteria.min_loss_pct) {
        candidates.push({ name: pos.name, reason: `亏损 ${lossPct.toFixed(1)}%` })
        continue
      }
      
      // 匹配条件 2: 属于不良板块 (简单关键词匹配)
      if (criteria.bad_sectors && criteria.bad_sectors.length > 0) {
        for (const bad of criteria.bad_sectors) {
          // 这里假设 name 或 ts_code 无法直接判断板块，暂且略过，或者以后端返回为准
          // 如果前端有板块字段可以使用
        }
      }
    }
    
    // 如果找到了合适的卖出标的
    if (candidates.length > 0) {
      // 取第一个
      const target = candidates[0]
      return {
        type: 'smart_replace',
        targetName: target.name,
        targetReason: target.reason,
        reason: criteria.reason || '优化持仓结构',
        message: `建议卖出 ${target.name}`
      }
    }
  }
  
  // 3. 兜底：显示通用建议
  if (props.stock.replacement_advice) {
    return {
      type: 'replace',
      message: props.stock.replacement_advice,
      shares: 0
    }
  }
  
  return {
    type: 'replace',
    message: '可用资金不足，建议调仓买入',
    shares: 0
  }
})

function getSignalClass(signal: string) {
  if (signal === '买入') return 'signal-buy'
  if (signal === '回避') return 'signal-sell'
  return 'signal-hold'
}

function getScoreColor(score: number) {
  if (score >= 80) return '#22c55e'
  if (score >= 60) return '#eab308'
  return '#ef4444'
}
</script>

<template>
  <div class="stock-card">
    <!-- Headers & Score (Keep Existing) -->
    <div class="card-header">
      <div class="header-left">
        <div class="rank-badge">{{ stock.rank }}</div>
        <div class="stock-main">
          <div class="stock-name-row">
            <span class="stock-name">{{ stock.name }}</span>
            <span class="stock-code">{{ stock.ts_code }}</span>
          </div>
          <div class="stock-sub">{{ stock.sector }}</div>
        </div>
      </div>
      <div class="header-right">
        <div class="signal-badge" :class="getSignalClass(stock.signal)">
          {{ stock.signal }}
        </div>
      </div>
    </div>

    <!-- 评分条 -->
    <div class="score-line">
      <div class="score-meta">
        <span class="score-label">推荐评分</span>
        <span class="score-num" :style="{ color: getScoreColor(stock.score) }">{{ stock.score }}</span>
      </div>
      <div class="score-track">
        <div class="score-bar" 
             :style="{ 
               width: `${stock.score}%`, 
               backgroundColor: getScoreColor(stock.score) 
             }">
        </div>
      </div>
    </div>

    <!-- 价格网格 -->
    <div class="price-grid">
      <div class="price-item">
        <span class="label">当前价</span>
        <span class="value main">¥{{ stock.current_price?.toFixed(2) }}</span>
      </div>
      <div class="price-item">
        <span class="label">建议买入</span>
        <span class="value buy">¥{{ (stock.buy_price || stock.current_price)?.toFixed(2) }}</span>
      </div>
      <div class="price-item">
        <span class="label">目标价</span>
        <span class="value target">¥{{ (stock.sell_price || stock.target_price)?.toFixed(2) }}</span>
      </div>
      <div class="price-item">
        <span class="label">止损价</span>
        <span class="value stop">¥{{ stock.stop_loss_price?.toFixed(2) }}</span>
      </div>
    </div>

    <!-- 交易行动建议 (Action Box) -->
    <div class="action-box">
      <div class="action-header">
        <span class="icon-dot"></span>
        <span class="action-title">交易建议</span>
      </div>
      
      <!-- 场景A：建议买入 -->
      <div v-if="tradeSuggestion.type === 'buy'" class="action-content buy-mode">
        <div class="suggestion-main">
          建议买入 <span class="highlight">{{ tradeSuggestion.shares }}</span> 股
        </div>
        <div class="suggestion-sub">
          预计金额 ¥{{ tradeSuggestion.amount?.toLocaleString() }} | 建议仓位 <span class="highlight">{{ stock.position_strategy?.initial_pct || 20 }}%</span>
        </div>
      </div>

      <!-- 场景B：智能换仓 (NEW) -->
      <div v-else-if="tradeSuggestion.type === 'smart_replace'" class="action-content replace-mode">
        <div class="suggestion-main text-warning">
          建议卖出 <span class="highlight-sell">{{ tradeSuggestion.targetName }}</span>
        </div>
        <div class="suggestion-sub">
          原因: {{ tradeSuggestion.targetReason }} ({{ tradeSuggestion.reason }})
        </div>
        <div class="suggestion-arrow">
           ↓ 调仓买入本股
        </div>
      </div>

      <!-- 场景C：通用换仓 -->
      <div v-else class="action-content replace-mode">
        <div class="suggestion-main text-warning">
          建议调仓
        </div>
        <div class="suggestion-desc">
          {{ tradeSuggestion.message }}
        </div>
      </div>
      
      <div class="strategy-mini" v-if="stock.position_strategy">
        <div class="mini-row">
          <span>策略: {{ stock.position_strategy.add_condition }}</span>
        </div>
      </div>
    </div>

    <!-- 逻辑与风险 -->
    <div class="analysis-grid">
      <div class="analysis-col">
        <div class="col-title">推荐逻辑</div>
        <ul class="bullet-list">
          <li v-for="(r, i) in stock.reasons" :key="i">{{ r }}</li>
        </ul>
      </div>
      <div class="analysis-col">
        <div class="col-title">风险因素</div>
        <ul class="bullet-list risk">
          <li v-for="(r, i) in stock.risk_factors" :key="i">{{ r }}</li>
        </ul>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* 保持原有样式，新增 highlight-sell */
.stock-card {
  background: linear-gradient(145deg, rgba(30, 30, 46, 0.9), rgba(20, 20, 35, 0.95));
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 12px;
  padding: 16px;
  margin-bottom: 16px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.2);
  transition: transform 0.2s ease, border-color 0.2s ease;
}

.stock-card:hover {
  transform: translateY(-2px);
  border-color: rgba(139, 92, 246, 0.4);
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 10px;
}

.rank-badge {
  background: linear-gradient(135deg, #6366f1, #8b5cf6);
  width: 24px;
  height: 24px;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
  font-size: 12px;
  color: white;
  box-shadow: 0 2px 4px rgba(99, 102, 241, 0.3);
}

.stock-main {
  display: flex;
  flex-direction: column;
}

.stock-name-row {
  display: flex;
  align-items: baseline;
  gap: 6px;
}

.stock-name {
  font-weight: 600;
  font-size: 15px;
  color: #fff;
}

.stock-code {
  font-size: 12px;
  color: #94a3b8;
  font-family: 'Monaco', 'Consolas', monospace;
}

.stock-sub {
  font-size: 11px;
  color: #64748b;
}

.signal-badge {
  padding: 4px 10px;
  border-radius: 12px;
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.5px;
  border: 1px solid transparent;
}

.signal-buy {
  background: rgba(34, 197, 94, 0.1);
  color: #4ade80;
  border-color: rgba(34, 197, 94, 0.2);
}

.signal-sell {
  background: rgba(239, 68, 68, 0.1);
  color: #f87171;
  border-color: rgba(239, 68, 68, 0.2);
}

.signal-hold {
  background: rgba(234, 179, 8, 0.1);
  color: #facc15;
  border-color: rgba(234, 179, 8, 0.2);
}

.score-line {
  margin-bottom: 14px;
}

.score-meta {
  display: flex;
  justify-content: space-between;
  font-size: 11px;
  margin-bottom: 4px;
  color: #94a3b8;
}

.score-num {
  font-weight: 700;
}

.score-track {
  height: 4px;
  background: rgba(255, 255, 255, 0.05);
  border-radius: 2px;
  overflow: hidden;
}

.score-bar {
  height: 100%;
  border-radius: 2px;
}

.price-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 8px;
  margin-bottom: 14px;
  background: rgba(255, 255, 255, 0.03);
  padding: 10px;
  border-radius: 8px;
}

.price-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 2px;
}

.price-item .label {
  font-size: 10px;
  color: #64748b;
}

.price-item .value {
  font-size: 13px;
  font-weight: 600;
  font-family: 'Roboto Mono', monospace;
}

.value.main { color: #f1f5f9; }
.value.buy { color: #60a5fa; }
.value.target { color: #4ade80; }
.value.stop { color: #f87171; }

.action-box {
  background: rgba(139, 92, 246, 0.05);
  border: 1px solid rgba(139, 92, 246, 0.15);
  border-radius: 8px;
  padding: 12px;
  margin-bottom: 14px;
}

.action-header {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-bottom: 8px;
}

.icon-dot {
  width: 6px;
  height: 6px;
  background: #8b5cf6;
  border-radius: 50%;
}

.action-title {
  font-size: 11px;
  font-weight: 600;
  color: #a78bfa;
  text-transform: uppercase;
}

.suggestion-main {
  font-size: 14px;
  font-weight: 500;
  color: #e2e8f0;
  margin-bottom: 2px;
}

.suggestion-main .highlight {
  color: #4ade80;
  font-weight: 700;
  font-size: 16px;
}

.suggestion-main .highlight-sell {
  color: #f87171; /* Red for sell */
  font-weight: 700;
}

.suggestion-main.text-warning {
  color: #facc15;
}

.suggestion-sub, .suggestion-desc {
  font-size: 11px;
  color: #94a3b8;
  line-height: 1.4;
}

.suggestion-arrow {
  margin-top: 4px;
  font-size: 11px;
  color: #4ade80;
}

.strategy-mini {
  margin-top: 8px;
  padding-top: 8px;
  border-top: 1px dashed rgba(139, 92, 246, 0.2);
}

.mini-row {
  font-size: 10px;
  color: #64748b;
  margin-bottom: 2px;
}

.analysis-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 16px;
}

.col-title {
  font-size: 11px;
  font-weight: 600;
  color: #94a3b8;
  margin-bottom: 6px;
  padding-left: 8px;
  border-left: 2px solid #334155;
}

.bullet-list {
  padding: 0;
  margin: 0;
  list-style: none;
}

.bullet-list li {
  position: relative;
  padding-left: 12px;
  font-size: 11px;
  color: #64748b;
  margin-bottom: 4px;
  line-height: 1.4;
}

.bullet-list li::before {
  content: '';
  position: absolute;
  left: 0;
  top: 5px;
  width: 4px;
  height: 4px;
  border-radius: 50%;
  background: #475569;
}

.bullet-list.risk li::before {
  background: #ef4444;
  opacity: 0.5;
}
</style>
