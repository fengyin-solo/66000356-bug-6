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
              :class="['cursor-pointer rounded border p-2 text-xs transition-all', store.activeSchema?.name === t.name ? 'border-cyan-500 bg-cyan-900/20' : isInvolved(t.name) ? 'border-orange-600/70 bg-orange-900/10 hover:border-orange-500' : 'border-slate-700 hover:border-slate-500']">
              <div class="flex justify-between items-center">
                <span class="font-bold text-slate-200">
                  {{ t.name }}
                  <span v-if="isInvolved(t.name)" class="ml-1 text-[10px] text-orange-400">● 查询中</span>
                </span>
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
          <div v-if="involvedTables.length" class="flex flex-wrap gap-2 mb-3">
            <button v-for="t in involvedTables" :key="t" type="button" @click="toggleSchema(t)"
              :class="['px-2 py-1 rounded border text-xs font-mono transition-all', store.activeSchema?.name === t ? 'border-cyan-500 bg-cyan-900/30 text-cyan-300' : 'border-slate-600 bg-slate-900 text-slate-300 hover:border-cyan-500']">
              {{ t }}
              <span class="ml-1 text-slate-500">{{ tableRowCount(t).toLocaleString() }} 行</span>
            </button>
          </div>
          <div v-else class="mb-3 text-xs text-slate-500 bg-slate-900 border border-slate-700 rounded p-2">未从语句中解析到任何涉及表</div>
          <div ref="erWrapRef" class="relative w-full">
            <canvas ref="erCanvasRef" class="w-full bg-slate-900 rounded block" style="height:200px" @click="onCanvasClick"></canvas>
            <div v-if="involvedTables.length && !joinEdges.length" class="absolute inset-x-0 bottom-2 mx-auto w-fit max-w-[90%] text-xs text-slate-400 bg-slate-800/90 border border-slate-600 rounded px-3 py-1.5 pointer-events-none">
              该查询没有 JOIN 连接（共 {{ involvedTables.length }} 张表）
            </div>
          </div>
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
const erWrapRef = ref<HTMLDivElement | null>(null)

// 与结果面板/图表保持一致的涉及表清单
const involvedTables = computed(() => store.parsed?.tables ?? [])
// 真实连接两端（过滤掉端点缺失的连接）
const joinEdges = computed(() => (store.parsed?.joins ?? []).filter(j => involvedTables.value.includes(j.table) && involvedTables.value.includes(j.leftTable)))

function isInvolved(name: string) {
  return involvedTables.value.includes(name)
}
function tableRowCount(name: string) {
  return SCHEMA_TABLES.find(s => s.name === name)?.rowCount ?? 0
}
function toggleSchema(name: string) {
  const t = SCHEMA_TABLES.find(s => s.name === name)
  if (!t) return
  store.activeSchema = store.activeSchema?.name === name ? null : t
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

const BOX_W = 120
const BOX_H = 56
const PAD_X = 24
const ROW_GAP = 56
const MIN_H = 200

// 命中检测用的表框坐标（CSS 像素），非响应式
let boxRects: { name: string; x: number; y: number }[] = []
let resizeObserver: ResizeObserver | null = null
let drawQueued = false

function roundRectPath(ctx: CanvasRenderingContext2D, x: number, y: number, w: number, h: number, r: number) {
  const rr = Math.min(r, w / 2, h / 2)
  ctx.beginPath()
  ctx.moveTo(x + rr, y)
  ctx.arcTo(x + w, y, x + w, y + h, rr)
  ctx.arcTo(x + w, y + h, x, y + h, rr)
  ctx.arcTo(x, y + h, x, y, rr)
  ctx.arcTo(x, y, x + w, y, rr)
  ctx.closePath()
}

// 求矩形中心连线与矩形边框的交点
function edgePoint(cx: number, cy: number, tx: number, ty: number) {
  const dx = tx - cx, dy = ty - cy
  if (dx === 0 && dy === 0) return { x: cx + BOX_W / 2, y: cy }
  const scale = 1 / Math.max(Math.abs(dx) / (BOX_W / 2), Math.abs(dy) / (BOX_H / 2))
  return { x: cx + dx * scale, y: cy + dy * scale }
}

function truncate(ctx: CanvasRenderingContext2D, text: string, maxW: number) {
  if (ctx.measureText(text).width <= maxW) return text
  const ell = '…'
  let lo = 0, hi = text.length
  while (lo < hi) {
    const mid = (lo + hi + 1) >> 1
    if (ctx.measureText(text.slice(0, mid) + ell).width <= maxW) lo = mid
    else hi = mid - 1
  }
  return text.slice(0, lo) + ell
}

function drawER() {
  drawQueued = false
  const canvas = erCanvasRef.value
  if (!canvas || !store.parsed) return
  const ctx = canvas.getContext('2d')
  if (!ctx) return
  const tables = involvedTables.value

  // 重绘前按当前容器尺寸校验画布（设备像素比适配，保证清晰）
  const cssW = Math.max(canvas.clientWidth || (erWrapRef.value?.clientWidth ?? 0), 1)
  const perRow = Math.max(1, Math.floor((cssW - PAD_X) / (BOX_W + PAD_X)))
  const rows = Math.max(1, Math.ceil(tables.length / perRow))
  const cssH = Math.max(MIN_H, rows * (BOX_H + ROW_GAP) + ROW_GAP)
  const dpr = window.devicePixelRatio || 1
  const W = Math.round(cssW * dpr)
  const H = Math.round(cssH * dpr)
  if (canvas.width !== W || canvas.height !== H) {
    canvas.width = W
    canvas.height = H
    canvas.style.height = cssH + 'px'
  }
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
  // 清理旧内容（必须在尺寸校验之后，否则改 width 会把画布清空）
  ctx.clearRect(0, 0, cssW, cssH)
  boxRects = []
  if (!tables.length) return

  const colGap = (cssW - PAD_X * 2 - perRow * BOX_W) / Math.max(perRow - 1, 1)
  const stepX = perRow > 1 ? BOX_W + colGap : 0
  const positions: Record<string, { x: number; y: number }> = {}
  tables.forEach((t, i) => {
    const col = i % perRow
    const row = Math.floor(i / perRow)
    const x = perRow === 1 ? cssW / 2 - BOX_W / 2 : PAD_X + col * stepX
    const y = ROW_GAP / 2 + row * (BOX_H + ROW_GAP)
    positions[t] = { x, y }
    boxRects.push({ name: t, x, y })
  })

  // 先画连接：真实两端 + 连接类型 + ON 条件
  joinEdges.value.forEach(j => {
    const a = positions[j.leftTable]
    const b = positions[j.table]
    if (!a || !b || j.leftTable === j.table) return
    const acx = a.x + BOX_W / 2, acy = a.y + BOX_H / 2
    const bcx = b.x + BOX_W / 2, bcy = b.y + BOX_H / 2
    const p1 = edgePoint(acx, acy, bcx, bcy)
    const p2 = edgePoint(bcx, bcy, acx, acy)

    ctx.beginPath()
    ctx.moveTo(p1.x, p1.y)
    ctx.lineTo(p2.x, p2.y)
    ctx.strokeStyle = '#f97316'
    ctx.lineWidth = 1.5
    ctx.setLineDash([5, 4])
    ctx.stroke()
    ctx.setLineDash([])

    // 目标端箭头
    const ang = Math.atan2(p2.y - p1.y, p2.x - p1.x)
    ctx.beginPath()
    ctx.moveTo(p2.x, p2.y)
    ctx.lineTo(p2.x - 9 * Math.cos(ang - Math.PI / 6), p2.y - 9 * Math.sin(ang - Math.PI / 6))
    ctx.lineTo(p2.x - 9 * Math.cos(ang + Math.PI / 6), p2.y - 9 * Math.sin(ang + Math.PI / 6))
    ctx.closePath()
    ctx.fillStyle = '#f97316'
    ctx.fill()

    // 连线中点的类型 + 条件标签（带底色，避免与连线文字重叠）
    const mx = (p1.x + p2.x) / 2
    const my = (p1.y + p2.y) / 2
    ctx.font = 'bold 10px monospace'
    const typeW = ctx.measureText(j.type).width
    ctx.font = '10px monospace'
    const condText = truncate(ctx, j.condition, 200)
    const condW = j.condition ? Math.min(ctx.measureText(condText).width, 200) : 0
    const pad = 6
    const labelW = Math.max(typeW, condW) + pad * 2
    const labelH = j.condition ? 30 : 16
    ctx.fillStyle = 'rgba(15, 23, 42, 0.92)'
    ctx.strokeStyle = '#f97316'
    ctx.lineWidth = 1
    roundRectPath(ctx, mx - labelW / 2, my - labelH / 2, labelW, labelH, 4)
    ctx.fill()
    ctx.stroke()
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillStyle = '#fdba74'
    ctx.font = 'bold 10px monospace'
    ctx.fillText(j.type, mx, j.condition ? my - 7 : my)
    if (j.condition) {
      ctx.fillStyle = '#94a3b8'
      ctx.font = '10px monospace'
      ctx.fillText(condText, mx, my + 6)
    }
  })

  // 再画表框（压在连线之上）
  tables.forEach(t => {
    const { x, y } = positions[t]
    const active = store.activeSchema?.name === t
    ctx.fillStyle = active ? '#164e63' : '#1e293b'
    ctx.strokeStyle = active ? '#22d3ee' : '#3b82f6'
    ctx.lineWidth = active ? 2.5 : 2
    roundRectPath(ctx, x, y, BOX_W, BOX_H, 6)
    ctx.fill()
    ctx.stroke()
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillStyle = active ? '#67e8f9' : '#06b6d4'
    ctx.font = 'bold 13px monospace'
    ctx.fillText(truncate(ctx, t, BOX_W - 12), x + BOX_W / 2, y + BOX_H / 2 - 9)
    const schema = SCHEMA_TABLES.find(s => s.name === t)
    if (schema) {
      ctx.fillStyle = '#64748b'
      ctx.font = '10px monospace'
      ctx.fillText(schema.rowCount.toLocaleString() + ' rows', x + BOX_W / 2, y + BOX_H / 2 + 11)
    }
  })
}

function scheduleDraw() {
  if (drawQueued) return
  drawQueued = true
  void nextTick(drawER)
}

function onCanvasClick(e: MouseEvent) {
  const canvas = erCanvasRef.value
  if (!canvas) return
  const rect = canvas.getBoundingClientRect()
  const x = e.clientX - rect.left
  const y = e.clientY - rect.top
  const hit = boxRects.find(b => x >= b.x && x <= b.x + BOX_W && y >= b.y && y <= b.y + BOX_H)
  if (hit) toggleSchema(hit.name)
}

onMounted(() => {
  store.analyze()
  scheduleDraw()
  if (erWrapRef.value && typeof ResizeObserver !== 'undefined') {
    resizeObserver = new ResizeObserver(() => scheduleDraw())
    resizeObserver.observe(erWrapRef.value)
  }
  window.addEventListener('resize', scheduleDraw)
})

onBeforeUnmount(() => {
  resizeObserver?.disconnect()
  window.removeEventListener('resize', scheduleDraw)
})

// 涉及表/连接或侧栏高亮变化时，重绘以保持图表与清单一致
watch(() => [store.parsed, store.activeSchema?.name], scheduleDraw, { deep: true })
</script>
