<script setup>
import { onMounted, ref, watch, computed } from 'vue'
import { Chart, ArcElement, Tooltip, Legend } from 'chart.js'
Chart.register(ArcElement, Tooltip, Legend)

const LS_KEY = 'calculadora_gastos_v1'

// Estado
const ingresoMensual = ref(0)
const descripcion = ref('')
const monto = ref('')
const color = ref('#2dd4bf') // color inicial por defecto
const gastos = ref([]) // [{id, desc, monto, color}]

const totalGastos = computed(() =>
  gastos.value.reduce((acc, g) => acc + Number(g.monto || 0), 0)
)

const restante = computed(() =>
  Math.max(0, Number(ingresoMensual.value || 0) - totalGastos.value)
)

const sobrepaso = computed(() =>
  Math.max(0, totalGastos.value - Number(ingresoMensual.value || 0))
)

// Persistencia (cargar)
onMounted(() => {
  const raw = localStorage.getItem(LS_KEY)
  if (raw) {
    try {
      const data = JSON.parse(raw)
      ingresoMensual.value = Number(data.ingresoMensual || 0)
      gastos.value = Array.isArray(data.gastos) ? data.gastos : []
    } catch (_) {}
  }
  initChart()
})

// Persistencia (guardar)
watch([ingresoMensual, gastos], () => {
  localStorage.setItem(
    LS_KEY,
    JSON.stringify({
      ingresoMensual: Number(ingresoMensual.value || 0),
      gastos: gastos.value
    })
  )
  updateChart()
}, { deep: true })

// Utilidades
function agregarGasto() {
  const m = Number(monto.value)
  if (!descripcion.value.trim() || isNaN(m) || m <= 0) return
  gastos.value.push({
    id: Date.now() + Math.random(),
    desc: descripcion.value.trim(),
    monto: m,
    color: color.value || randomColor()
  })
  // limpiar inputs de gasto (no ingreso mensual)
  descripcion.value = ''
  monto.value = ''
  color.value = randomColor()
}

function eliminarGasto(id) {
  gastos.value = gastos.value.filter(g => g.id !== id)
}

function randomColor() {
  // color pastel-ish
  const hue = Math.floor(Math.random() * 360)
  return `hsl(${hue} 70% 55%)`
}

// ---- Chart.js (dona) ----
let chart
const canvasRef = ref(null)

// Plugin para texto centrado
const centerTextPlugin = {
  id: 'centerText',
  afterDraw(chart, args, pluginOptions) {
    const { ctx, chartArea: { width, height } } = chart
    ctx.save()
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.font = '700 18px system-ui, -apple-system, Segoe UI, Roboto'
    const r = restante.value.toLocaleString()
    const t = `Restante`
    ctx.fillStyle = '#6b7280'
    ctx.fillText(t, width / 2, height / 2 - 12)
    ctx.fillStyle = '#111827'
    ctx.font = '700 22px system-ui, -apple-system, Segoe UI, Roboto'
    ctx.fillText(`$ ${r}`, width / 2, height / 2 + 12)
    ctx.restore()
  }
}
Chart.register(centerTextPlugin)

function chartData() {
  const values = gastos.value.map(g => Number(g.monto))
  const colors = gastos.value.map(g => g.color)
  // el segmento “restante” en gris claro
  values.push(restante.value)
  colors.push('#e5e7eb')
  const labels = [...gastos.value.map(g => g.desc), 'Restante']
  return { values, colors, labels }
}

function initChart() {
  const ctx = canvasRef.value?.getContext('2d')
  if (!ctx) return
  const { values, colors, labels } = chartData()
  chart = new Chart(ctx, {
    type: 'doughnut',
    data: {
      labels,
      datasets: [
        {
          data: values,
          backgroundColor: colors,
          borderColor: '#ffffff',
          borderWidth: 2,
          hoverOffset: 6
        }
      ]
    },
    options: {
      cutout: '70%',
      plugins: {
        legend: { position: 'right' },
        tooltip: {
          callbacks: {
            label: (ctx) => {
              const val = Number(ctx.parsed)
              return `${ctx.label}: $ ${val.toLocaleString()}`
            }
          }
        }
      }
    }
  })
}

function updateChart() {
  if (!chart) return
  const { values, colors, labels } = chartData()
  chart.data.labels = labels
  chart.data.datasets[0].data = values
  chart.data.datasets[0].backgroundColor = colors
  chart.update()
}
</script>

<template>
  <div class="container">
    <h1>Calculadora de Gastos</h1>

    <section class="card">
      <label class="row">
        <span>Dinero mensual (neto)</span>
        <input type="number" min="0" step="0.01" v-model.number="ingresoMensual" placeholder="Ej. 10000" />
      </label>

      <div class="stats">
        <div>
          <small>Saldo inicial</small>
          <div class="num">$ {{ Number(ingresoMensual || 0).toLocaleString() }}</div>
        </div>
        <div>
          <small>Total gastos</small>
          <div class="num rojo">$ {{ totalGastos.toLocaleString() }}</div>
        </div>
        <div v-if="sobrepaso === 0">
          <small>Restante</small>
          <div class="num verde">$ {{ restante.toLocaleString() }}</div>
        </div>
        <div v-else>
          <small>Sobrepaso</small>
          <div class="num rojo">$ {{ sobrepaso.toLocaleString() }}</div>
        </div>
      </div>
    </section>

    <section class="layout">
      <div class="left card">
        <h2>Agregar gasto</h2>
        <form @submit.prevent="agregarGasto" class="form">
          <input v-model="descripcion" placeholder="Descripción (p. ej. Renta)" />
          <input v-model="monto" type="number" step="0.01" min="0" placeholder="Monto" />
          <input v-model="color" type="color" title="Color del gasto" />
          <button type="submit">Añadir</button>
        </form>

        <h3>Gastos</h3>
        <ul class="lista">
          <li v-for="g in gastos" :key="g.id" class="item">
            <span class="chip" :style="{ background: g.color }"></span>
            <span class="desc">{{ g.desc }}</span>
            <span class="monto">$ {{ Number(g.monto).toLocaleString() }}</span>
            <button class="del" @click="eliminarGasto(g.id)">✕</button>
          </li>
          <li v-if="gastos.length === 0" class="mute">Sin gastos aún</li>
        </ul>
      </div>

      <div class="right card">
        <h2>Distribución</h2>
        <canvas ref="canvasRef" height="280"></canvas>
        <p class="note">
          Cada color representa un gasto. El segmento gris es el dinero restante del mes.
        </p>
      </div>
    </section>
  </div>
</template>

<style scoped>
.container { max-width: 1100px; margin: 24px auto; padding: 0 16px; }
h1 { font-size: 28px; margin-bottom: 12px; }
.card { background: #fff; border: 1px solid #e5e7eb; border-radius: 16px; padding: 16px; box-shadow: 0 1px 3px rgba(0,0,0,.05); }

.row { display: flex; gap: 12px; align-items: center; }
.row input[type="number"] { width: 220px; }

.stats { display: grid; grid-template-columns: repeat(3,1fr); gap: 16px; margin-top: 12px; }
.stats .num { font-weight: 700; font-size: 20px; }
.stats .verde { color: #059669; }
.stats .rojo { color: #b91c1c; }

.layout { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; margin-top: 16px; }
.form { display: grid; grid-template-columns: 1fr 140px 60px 120px; gap: 8px; }
.form input, .form button { padding: 10px; border: 1px solid #e5e7eb; border-radius: 10px; }
.form button { background: #111827; color: #fff; font-weight: 600; cursor: pointer; }

.lista { list-style: none; padding: 0; margin: 10px 0 0; }
.item { display: grid; grid-template-columns: 18px 1fr auto 36px; gap: 8px; align-items: center;
  padding: 8px; border-bottom: 1px dashed #e5e7eb; }
.chip { width: 14px; height: 14px; border-radius: 50%; display: inline-block; border: 2px solid #fff; box-shadow: 0 0 0 1px #e5e7eb; }
.desc { font-weight: 600; }
.monto { font-variant-numeric: tabular-nums; }
.del { background: transparent; border: none; font-size: 18px; cursor: pointer; color: #6b7280; }
.mute { color: #9ca3af; padding: 8px; }
.right .note { color: #6b7280; font-size: 13px; margin-top: 8px; }

/* Responsive */
@media (max-width: 900px) {
  .layout { grid-template-columns: 1fr; }
  .stats { grid-template-columns: 1fr 1fr; }
}
</style>
