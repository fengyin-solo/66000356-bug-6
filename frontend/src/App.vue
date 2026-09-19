<template>
  <div class="min-h-screen bg-slate-900 text-slate-200">
    <header class="border-b border-slate-700 px-6 py-4">
      <h1 class="text-2xl font-bold text-cyan-400">SQL 查询可视化与执行计划分析器</h1>
      <p class="text-sm text-slate-500 mt-1">SQL语法解析 · 执行计划树 · ER图 · 复杂度评分 · 优化建议</p>
    </header>
    <div class="flex flex-col lg:flex-row gap-4 p-4">
      <div class="lg:w-2/5 space-y-4">
        <div class="bg-slate-800 rounded-lg p-4 border border-slate-700">
          <div class="flex items-center justify-between mb-3">
            <h3 class="text-sm font-bold text-slate-400">SQL 编辑器</h3>
            <div class="flex gap-2">
              <select @change="(e) => { store.sql = SQL_TEMPLATES[+(e.target as HTMLSelectElement).value].sql }" class="text-xs bg-slate-900 border border-slate-600 rounded px-2 py-1 text-slate-300">
                <option v-for="(t, i) in SQL_TEMPLATES" :key="i" :value="i">{{ t.name }}</option>
              </select>
            </div>
          </div>
          <textarea v-model="store.sql" rows="12" class="w-full bg-slate-900 border border-slate-600 rounded px-3 py-2 text-sm font-mono text-green-400 focus:outline-none focus:border-cyan-500 resize-none"></textarea>
          <button @click="store.analyze" class="w-full mt-3 py-2 bg-cyan-600 hover:bg-cyan-500 rounded text-sm font-bold">分析查询</button>
        </div>
        <div class="bg-slate-800 rounded-lg p-4 border border-slate-700">
          <h3 class="text-sm font-bold text-slate-400 mb-3">数据库 Schema</h3>
          <div class="space-y-2">
            <div v-for="t in SCHEMA_TABLES" :key="t.name" @click="store.activeSchema = store.activeSchema?.name === t.name ? null : t"
              :class="['cursor-pointer rounded border p-2 text-xs transition-all', store.activeSchema?.name === t.name ? 'border-cyan-500 bg-cyan-900/20' : 'border-slate-700 hover:border-slate-500']">
              <div class="flex justify-between items-center">
                <span class="font-bold text-slate-200">{{ t.name }}</span>
                <span class="text-slate-500">{{ t.rowCount.toLocaleString() }} 行</span>
              </div>
              <div v-if="store.activeSchema?.name === t.name" class="mt-2 space-y-0.5">
                <div v-for="c in t.columns" :key="c.name" class="flex gap-2">
                  <span :class="c.pk ? 'text-yellow-400' : c.fk ? 'text-blue-400' : 'text-slate-400'">{{ c.pk ? '🔑 ' : c.fk ? '🔗 ' : '  ' }}{{ c.name }}</span>
                  <span class="text-slate-600">{{ c.type }}</span>
                  <span v-if="c.fk" class="text-blue-600">→ {{ c.fk }}</span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
      <div class="lg:w-3/5 space-y-4">
        <div v-if="store.parsed" class="bg-slate-800 rounded-lg p-4 border border-slate-700">
          <h3 class="text-sm font-bold text-slate-400 mb-3">查询解析结果</h3>
          <div class="grid grid-cols-4 gap-3 text-sm mb-4">
            <div class="bg-slate-900 rounded p-2 text-center"><div class="text-xs text-slate-500 mb-1">类型</div><div class="text-cyan-400 font-bold">{{ store.parsed.type }}</div></div>
            <div class="bg-slate-900 rounded p-2 text-center"><div class="text-xs text-slate-500 mb-1">复杂度</div><div class="font-bold" :class="store.complexityLabel.color">{{ store.complexityLabel.label }}</div></div>
            <div class="bg-slate-900 rounded p-2 text-center"><div class="text-xs text-slate-500 mb-1">JOIN数</div><div class="text-orange-400 font-bold">{{ store.parsed.joins.length }}</div></div>
            <div class="bg-slate-900 rounded p-2 text-center"><div class="text-xs text-slate-500 mb-1">预估行数</div><div class="text-purple-400 font-bold">{{ store.parsed.estimatedCost }}</div></div>
          </div>
          <div v-if="store.parsed.suggestions.length" class="space-y-1">
            <div class="text-xs text-slate-500 mb-1">优化建议</div>
            <div v-for="(s, i) in store.parsed.suggestions" :key="i" class="text-xs flex items-start gap-2 bg-orange-900/30 border border-orange-700 rounded p-2">
              <span class="text-orange-400">⚠</span><span class="text-orange-300">{{ s }}</span>
            </div>
          </div>
          <div v-else class="text-xs text-green-400 bg-green-900/20 border border-green-700 rounded p-2">✓ 未发现明显性能问题</div>
        </div>
        <div v-if="store.plan" class="bg-slate-800 rounded-lg p-4 border border-slate-700">
          <h3 class="text-sm font-bold text-slate-400 mb-3">执行计划树</h3>
          <div class="overflow-x-auto">
            <div class="font-mono text-xs text-slate-300 space-y-1">
              <PlanNode :node="store.plan" :depth="0" />
            </div>
          </div>
        </div>
        <div v-if="store.parsed" class="bg-slate-800 rounded-lg p-4 border border-slate-700">
          <h3 class="text-sm font-bold text-slate-400 mb-3">涉及表与关联关系</h3>
          <div class="flex flex-wrap gap-2 mb-3">
            <button v-for="t in involvedTables" :key="t" @click="toggleSchema(t)"
              :class="['px-2 py-1 rounded text-xs font-mono border transition-all',
                store.activeSchema?.name === t ? 'border-cyan-400 bg-cyan-900/30 text-cyan-300' : 'border-slate-600 bg-slate-900 text-slate-300 hover:border-slate-400']">
              {{ t }}
            </button>
            <span v-if="!involvedTables.length" class="text-xs text-slate-500">未解析到涉及的表</span>
          </div>
          <canvas ref="erCanvasRef" class="w-full bg-slate-900 rounded block" style="height:220px"></canvas>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch, onMounted, onBeforeUnmount, nextTick, defineComponent, h } from 'vue'
import { useSQLStore, SQL_TEMPLATES, SCHEMA_TABLES } from './store/sql'

const store = useSQLStore()
const erCanvasRef = ref<HTMLCanvasElement | null>(null)

// 涉及表清单：与关系图共用同一份去重后的数据，保证两边一致
const involvedTables = computed(() => {
  if (!store.parsed) return [] as string[]
  return Array.from(new Set(store.parsed.tables))
})

function toggleSchema(name: string) {
  const schema = SCHEMA_TABLES.find(s => s.name === name)
  store.activeSchema = store.activeSchema?.name === name ? null : schema ?? null
}

const PlanNode = defineComponent({
  props: { node: Object, depth: Number },
  setup(props) {
    return () => {
      if (!props.node) return null
      const n = props.node as any
      const indent = '  '.repeat(props.depth || 0)
      const opColor = n.operation.includes('Scan') ? '#22c55e' : n.operation.includes('Join') ? '#f97316' : n.operation.includes('Sort') ? '#8b5cf6' : '#06b6d4'
      return h('div', [
        h('div', { style: `padding-left: ${(props.depth || 0) * 20}px` }, [
          h('span', { style: 'color: #475569' }, indent.replace(/\s\s/g, '│ ').replace(/│ $/, '└─')),
          h('span', { style: `color: ${opColor}; font-weight: bold` }, n.operation),
          n.table ? h('span', { style: 'color: #94a3b8' }, ` on ${n.table}`) : null,
          n.index ? h('span', { style: 'color: #eab308' }, ` [${n.index}]`) : null,
          h('span', { style: 'color: #64748b' }, ` cost=${n.cost.toFixed(1)} rows=${n.rows}`),
        ]),
        ...(n.children || []).map((child: any) => h(PlanNode, { node: child, depth: (props.depth || 0) + 1 }))
      ])
    }
  }
})

const BOX_W = 108
const BOX_H = 54
const ROW_H = 96

interface Pt { x: number; y: number }

// 表框边缘与连线的交点，避免连线穿到表框底下
function edgePoint(from: Pt, to: Pt): Pt {
  const dx = to.x - from.x
  const dy = to.y - from.y
  if (!dx && !dy) return { ...from }
  const sx = dx !== 0 ? (BOX_W / 2) / Math.abs(dx) : Infinity
  const sy = dy !== 0 ? (BOX_H / 2) / Math.abs(dy) : Infinity
  const s = Math.min(sx, sy)
  return { x: from.x + dx * s, y: from.y + dy * s }
}

// 连接标记：连接类型 + ON 条件，画在带底色的标签里，避免和连线混在一起
function drawJoinLabel(ctx: CanvasRenderingContext2D, x: number, y: number, j: { type: string; condition: string }) {
  const typeText = (j.type || 'INNER').toUpperCase() + ' JOIN'
  let cond = (j.condition || '').trim()
  if (cond.length > 34) cond = cond.slice(0, 31) + '…'
  ctx.font = 'bold 9px monospace'
  const w1 = ctx.measureText(typeText).width
  ctx.font = '9px monospace'
  const w2 = cond ? ctx.measureText(cond).width : 0
  const w = Math.max(w1, w2) + 14
  const h = cond ? 28 : 15
  ctx.fillStyle = 'rgba(2, 6, 23, 0.92)'
  ctx.strokeStyle = '#f97316'
  ctx.lineWidth = 1
  ctx.beginPath()
  ctx.roundRect(x - w / 2, y - h / 2, w, h, 4)
  ctx.fill()
  ctx.stroke()
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'
  ctx.fillStyle = '#fb923c'
  ctx.font = 'bold 9px monospace'
  ctx.fillText(typeText, x, cond ? y - 6 : y)
  if (cond) {
    ctx.fillStyle = '#cbd5e1'
    ctx.font = '9px monospace'
    ctx.fillText(cond, x, y + 7)
  }
}

function drawER() {
  const canvas = erCanvasRef.value
  if (!canvas || !store.parsed) return
  const ctx = canvas.getContext('2d')
  if (!ctx) return

  const tables = involvedTables.value
  const joins = store.parsed.joins

  // 重绘前按当前容器尺寸校验画布：容器宽度变化时重新计算布局与画布高度
  const cssW = canvas.clientWidth
  if (!cssW) return
  // 每行最多放的表框数：保证均分后间距不小于 BOX_W + 24，避免相邻表框重叠
  const MIN_SPACING = BOX_W + 24
  let perRow = Math.max(1, Math.floor(cssW / MIN_SPACING))
  while (perRow > 1 && cssW / (perRow + 1) < MIN_SPACING) perRow--
  const rows = Math.max(1, Math.ceil(tables.length / perRow))
  const cssH = tables.length ? rows * ROW_H + 24 : 120
  const dpr = window.devicePixelRatio || 1
  const needW = Math.round(cssW * dpr)
  const needH = Math.round(cssH * dpr)
  if (canvas.width !== needW || canvas.height !== needH) {
    canvas.width = needW
    canvas.height = needH
  }
  if (canvas.style.height !== cssH + 'px') canvas.style.height = cssH + 'px'
  // 清理旧内容（尺寸未变时画布不会自动清空，需要手动清）
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
  ctx.clearRect(0, 0, cssW, cssH)

  // 空态：没有解析到任何表
  if (!tables.length) {
    ctx.fillStyle = '#475569'
    ctx.font = '12px sans-serif'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText('未解析到涉及的表', cssW / 2, cssH / 2)
    return
  }

  // 布局：一行放不下时自动换行，每行水平均分
  const positions: Record<string, Pt> = {}
  tables.forEach((t, i) => {
    const row = Math.floor(i / perRow)
    const col = i % perRow
    const inRow = Math.min(perRow, tables.length - row * perRow)
    const spacing = cssW / (inRow + 1)
    positions[t] = { x: spacing * (col + 1), y: row * ROW_H + ROW_H / 2 + 12 }
  })

  const active = store.activeSchema?.name

  // 连线：按语句真实的连接两端绘制
  const labels: { x: number; y: number; j: (typeof joins)[number] }[] = []
  joins.forEach((j, idx) => {
    const a = positions[j.fromTable || ''] || positions[tables[0]]
    const b = positions[j.toTable || ''] || positions[j.table]
    if (!a || !b) return
    ctx.strokeStyle = '#f97316'
    ctx.lineWidth = 1.5
    ctx.setLineDash([5, 4])
    ctx.beginPath()
    if (a === b) {
      // 自连接：在表框上方画环，标记放在表框下方避免超出画布
      ctx.arc(a.x, a.y - BOX_H / 2 - 14, 14, 0, Math.PI * 2)
      ctx.stroke()
      ctx.setLineDash([])
      labels.push({ x: a.x, y: a.y + BOX_H / 2 + 18, j })
    } else {
      const p1 = edgePoint(a, b)
      const p2 = edgePoint(b, a)
      ctx.moveTo(p1.x, p1.y)
      ctx.lineTo(p2.x, p2.y)
      ctx.stroke()
      ctx.setLineDash([])
      labels.push({ x: (p1.x + p2.x) / 2, y: (p1.y + p2.y) / 2 - 8 - (idx % 2) * 22, j })
    }
  })

  // 表框：侧栏展开的表同步高亮
  tables.forEach(t => {
    const pos = positions[t]
    const isActive = active === t
    ctx.save()
    if (isActive) {
      ctx.shadowColor = '#22d3ee'
      ctx.shadowBlur = 12
    }
    ctx.fillStyle = isActive ? '#164e63' : '#1e293b'
    ctx.strokeStyle = isActive ? '#22d3ee' : '#3b82f6'
    ctx.lineWidth = isActive ? 2.5 : 1.5
    ctx.beginPath()
    ctx.roundRect(pos.x - BOX_W / 2, pos.y - BOX_H / 2, BOX_W, BOX_H, 6)
    ctx.fill()
    ctx.stroke()
    ctx.restore()
    ctx.fillStyle = isActive ? '#a5f3fc' : '#06b6d4'
    ctx.font = 'bold 12px monospace'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText(t, pos.x, pos.y - 9, BOX_W - 12)
    const schema = SCHEMA_TABLES.find(s => s.name === t)
    if (schema) {
      ctx.fillStyle = '#64748b'
      ctx.font = '10px monospace'
      ctx.fillText(schema.rowCount.toLocaleString() + ' rows', pos.x, pos.y + 11)
    }
  })

  // 连接标记最后画，保证不被表框遮住
  labels.forEach(l => drawJoinLabel(ctx, l.x, l.y, l.j))

  // 空态：没有任何连接时给出说明
  if (!joins.length) {
    ctx.fillStyle = '#475569'
    ctx.font = '11px sans-serif'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText('当前语句没有表连接（无 JOIN），仅展示涉及的表', cssW / 2, 14)
  }
}

let resizeObserver: ResizeObserver | null = null

onMounted(async () => {
  store.analyze()
  await nextTick()
  drawER()
  // 容器宽度变化（窗口缩放、布局切换）时按新尺寸重绘，避免标记错位
  if (erCanvasRef.value) {
    resizeObserver = new ResizeObserver(() => drawER())
    resizeObserver.observe(erCanvasRef.value)
  }
})
onBeforeUnmount(() => resizeObserver?.disconnect())

// 涉及表 / 连接变化后重绘，图表与清单保持一致；侧栏展开的表变化后同步高亮
watch(() => store.parsed, () => nextTick(drawER), { deep: true })
watch(() => store.activeSchema, () => drawER())
</script>
