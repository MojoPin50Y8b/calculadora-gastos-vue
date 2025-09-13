<script setup>
import { onMounted, ref, watch, computed } from 'vue'
import { Chart, ArcElement, Tooltip, Legend, DoughnutController } from 'chart.js'
Chart.register(ArcElement, Tooltip, Legend, DoughnutController)

// Excel / PDF
import * as XLSX from 'xlsx'
import jsPDF from 'jspdf'
import autoTable from 'jspdf-autotable'

const LS_KEY = 'calculadora_gastos_v1'

// ---------------- Estado ----------------
const ingresoMensual = ref(0)
const descripcion = ref('')
const monto = ref('')
const color = ref('#2dd4bf')
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

// ---------------- Persistencia (cargar) ----------------
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

// ---------------- Persistencia (guardar) ----------------
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

// ---------------- Utilidades ----------------
function agregarGasto() {
  const m = Number(monto.value)
  if (!descripcion.value.trim() || isNaN(m) || m <= 0) return
  gastos.value.push({
    id: Date.now() + Math.random(),
    desc: descripcion.value.trim(),
    monto: m,
    color: color.value || randomColor()
  })
  descripcion.value = ''
  monto.value = ''
  color.value = randomColor()
}
function eliminarGasto(id) {
  gastos.value = gastos.value.filter(g => g.id !== id)
}
function randomColor() {
  const hue = Math.floor(Math.random() * 360)
  return `hsl(${hue} 70% 55%)`
}

// ---------------- Chart.js (dona) ----------------
let chart
const canvasRef = ref(null)

const centerTextPlugin = {
  id: 'centerText',
  afterDraw(chart) {
    const { ctx, chartArea: { width, height } } = chart
    ctx.save()
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.font = '700 18px system-ui, -apple-system, Segoe UI, Roboto'
    const r = restante.value.toLocaleString()
    ctx.fillStyle = '#6b7280'
    ctx.fillText('Restante', width / 2, height / 2 - 12)
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

// ---------------- Exportar / Importar ----------------
function getGastosPlanos() {
  return gastos.value.map(g => ({
    Descripción: g.desc,
    Monto: Number(g.monto || 0),
    Color: g.color || '#cccccc'
  }))
}
function applyRowsToGastos(rows = []) {
  const sane = rows
    .filter(r => (r.Descripción || r.descripcion) && !isNaN(Number(r.Monto)))
    .map((r, i) => ({
      id: Date.now() + i + Math.random(),
      desc: String(r.Descripción ?? r.descripcion).trim(),
      monto: Number(r.Monto),
      color: String(r.Color || '').trim() || randomColor()
    }))
  gastos.value = sane
}
async function exportarExcel() {
  const wb = XLSX.utils.book_new()
  const wsGastos = XLSX.utils.json_to_sheet(getGastosPlanos())
  XLSX.utils.book_append_sheet(wb, wsGastos, 'Gastos')
  const wsResumen = XLSX.utils.aoa_to_sheet([
    ['IngresoMensual', Number(ingresoMensual.value || 0)],
    ['TotalGastos', totalGastos.value],
    ['Restante', restante.value],
    ['Sobrepaso', sobrepaso.value]
  ])
  XLSX.utils.book_append_sheet(wb, wsResumen, 'Resumen')
  XLSX.writeFile(wb, 'reporte_gastos.xlsx')
}
async function importarExcel(archivo) {
  if (!archivo) return
  const buf = await archivo.arrayBuffer()
  const wb = XLSX.read(buf, { type: 'array' })
  const wsResumen = wb.Sheets['Resumen']
  if (wsResumen) {
    const aoa = XLSX.utils.sheet_to_json(wsResumen, { header: 1 })
    const filaIngreso = (aoa || []).find(r => (r?.[0] + '').toLowerCase() === 'ingresomensual')
    if (filaIngreso && !isNaN(Number(filaIngreso[1]))) {
      ingresoMensual.value = Number(filaIngreso[1])
    }
  }
  const wsGastos = wb.Sheets['Gastos'] || wb.Sheets[wb.SheetNames[0]]
  const rows = XLSX.utils.sheet_to_json(wsGastos ?? {})
  applyRowsToGastos(rows)
}
function exportarPDF() {
  const doc = new jsPDF({ unit: 'pt', format: 'a4' })
  const marginX = 40
  let y = 40

  doc.setFont('helvetica', 'bold'); doc.setFontSize(18)
  doc.text('Reporte de Gastos', marginX, y); y += 22

  doc.setFontSize(12); doc.setFont('helvetica', 'normal')
  doc.text(`Ingreso mensual: $ ${Number(ingresoMensual.value || 0).toLocaleString()}`, marginX, y); y += 16
  doc.text(`Total gastos:    $ ${totalGastos.value.toLocaleString()}`, marginX, y); y += 16
  if (sobrepaso.value === 0) {
    doc.text(`Restante:        $ ${restante.value.toLocaleString()}`, marginX, y)
  } else {
    doc.text(`Sobrepaso:       $ ${sobrepaso.value.toLocaleString()}`, marginX, y)
  }
  y += 24

  const canvas = canvasRef.value
  if (canvas) {
    const img = canvas.toDataURL('image/png', 1.0)
    const imgW = 240
    const imgH = (canvas.height / canvas.width) * imgW
    doc.addImage(img, 'PNG', marginX, y, imgW, imgH)
  }

  const tableStartY = y
  const leftForTable = canvas ? 300 : marginX
  autoTable(doc, {
    startY: tableStartY,
    margin: { left: leftForTable },
    styles: { fontSize: 10 },
    head: [['Descripción', 'Monto', 'Color']],
    body: gastos.value.map(g => [
      g.desc,
      `$ ${Number(g.monto).toLocaleString()}`,
      g.color
    ])
  })

  doc.setFontSize(9)
  doc.text(`Generado: ${new Date().toLocaleString()}`, marginX, 820)
  doc.save('reporte_gastos.pdf')
}
function onFileChange(e) {
  const file = e?.target?.files?.[0]
  importarExcel(file)
  if (e?.target) e.target.value = ''
}
</script>

<template>
  <div class="container">
    <h1>Calculadora de Gastos</h1>

    <!-- Top: 2 columnas (card de Dinero y card de Botones) -->
    <section class="top-grid">
      <!-- Card: Dinero mensual -->
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

      <!-- Card: Botones -->
      <section class="card actions-card">
        <div class="actions">
          <button class="btn btn-pdf" @click="exportarPDF" title="Descargar PDF">
            <!-- icono PDF -->
            <svg width="18" height="18" viewBox="0 0 24 24" aria-hidden="true"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z" fill="currentColor" opacity=".15"/><path d="M14 2v6a2 2 0 0 0 2 2h6" fill="none" stroke="currentColor" stroke-width="2"/><path d="M7 15h2.5a1.5 1.5 0 0 0 0-3H7v6M13 12h1.5a1.5 1.5 0 0 1 0 3H13v3M18 12h2" fill="none" stroke="currentColor" stroke-width="2"/></svg>
            <span>Descargar PDF</span>
          </button>

          <button class="btn btn-excel" @click="exportarExcel" title="Descargar Excel">
            <!-- icono Excel -->
            <svg width="18" height="18" viewBox="0 0 24 24" aria-hidden="true"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z" fill="currentColor" opacity=".15"/><path d="M16 12l-3 4m0-4l3 4m-5-4l-3 4m0-4l3 4" fill="none" stroke="currentColor" stroke-width="2"/></svg>
            <span>Descargar Excel</span>
          </button>

          <label class="btn btn-excel outline">
            <svg width="18" height="18" viewBox="0 0 24 24" aria-hidden="true"><path d="M12 3v12m0 0l-4-4m4 4l4-4M4 19h16" fill="none" stroke="currentColor" stroke-width="2"/></svg>
            <span>Cargar Excel</span>
            <input type="file" accept=".xlsx,.xls" @change="onFileChange">
          </label>
        </div>
      </section>
    </section>

    <!-- Layout principal -->
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

    <!-- Footer -->
    <footer class="site-footer">
      <span>© {{ new Date().getFullYear() }} Calculadora de Gastos — Todos los derechos reservados. MojoPin50Y8b.</span>
    </footer>
  </div>
</template>

<style scoped>
.container { max-width: 1100px; margin: 24px auto; padding: 0 16px; }
h1 { font-size: 28px; margin-bottom: 12px; }
.card { background: #fff; border: 1px solid #e5e7eb; border-radius: 16px; padding: 16px; box-shadow: 0 1px 3px rgba(0,0,0,.05); }

/* Top grid: card de dinero + card de botones */
.top-grid {
  display: grid;
  grid-template-columns: 2fr 1fr;
  gap: 16px;
  margin-bottom: 16px;
}

/* Formato del card de acciones */
.actions-card { display: flex; align-items: center; }
.actions { display:flex; flex-wrap:wrap; gap:10px; width:100%; justify-content:flex-start; }
.btn {
  display:inline-flex; align-items:center; gap:8px;
  padding:10px 14px; font-weight:700; border-radius:12px;
  border:1px solid transparent; cursor:pointer; user-select:none;
  color:#fff; background:#111827; transition: transform .06s ease, box-shadow .2s ease;
  position:relative;
}
.btn svg { display:block }
.btn:hover { box-shadow: 0 6px 14px rgba(0,0,0,.08); transform: translateY(-1px); }
.btn:active { transform: translateY(0); }

/* Colores */
.btn-pdf { background:#b91c1c; }           /* rojo */
.btn-pdf:hover { background:#991b1b; }
.btn-excel { background:#059669; }          /* verde */
.btn-excel:hover { background:#047857; }

/* Label-file como botón */
.btn.outline {
  background:transparent; color:#059669; border-color:#059669;
}
.btn.outline:hover { background:rgba(5,150,105,.08); }
.btn.outline input[type="file"]{
  position:absolute; inset:0; opacity:0; cursor:pointer;
}

/* Campos superiores */
.row { display: flex; gap: 12px; align-items: center; }
.row input[type="number"] { width: 220px; }

.stats { display: grid; grid-template-columns: repeat(3,1fr); gap: 16px; margin-top: 12px; }
.stats .num { font-weight: 700; font-size: 20px; }
.stats .verde { color: #059669; }
.stats .rojo { color: #b91c1c; }

/* Layout principal */
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

/* Footer */
.site-footer { margin: 28px 0 10px; text-align:center; color:#6b7280; font-size: 12px; }

/* Responsive */
@media (max-width: 900px) {
  .layout { grid-template-columns: 1fr; }
  .stats { grid-template-columns: 1fr 1fr; }
  .top-grid { grid-template-columns: 1fr; }
  /* card de botones debajo y alineado a la derecha */
  .actions-card { justify-content: flex-end; }
}
</style>
