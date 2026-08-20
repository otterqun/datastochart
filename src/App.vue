<script setup>
import { ref, onMounted, watch } from 'vue'
import Chart from 'chart.js/auto'
import * as XLSX from 'xlsx'

// 1. STATE PENGURUSAN DATA (MULTI-DATASET)
const chartTitle = ref('Perbandingan Jualan 2025 vs 2026')
const chartType = ref('bar')

// Metadata untuk kumpulan data (Lajur / Dataset)
const colorPalette = ['#4f46e5', '#ec4899', '#10b981', '#f59e0b', '#3b82f6', '#8b5cf6', '#ef4444']
const datasetMeta = ref([
  { id: 'ds1', name: 'Tahun 2025', color: '#4f46e5' },
  { id: 'ds2', name: 'Tahun 2026', color: '#ec4899' }
])
let nextDsId = 3

// Data baris (Label dan nilai-nilai untuk setiap dataset)
const dataRows = ref([
  { id: 1, label: 'Januari', values: { ds1: 12, ds2: 15 } },
  { id: 2, label: 'Februari', values: { ds1: 19, ds2: 25 } },
  { id: 3, label: 'Mac', values: { ds1: 3, ds2: 8 } },
])
let nextRowId = 4

// State untuk fungsi Upload & Mapping Excel
const rawExcelData = ref([])
const excelHeaders = ref([])
const selectedLabelCol = ref('')
const selectedDataCols = ref([]) // <-- Sekarang berbentuk array (multi-select)
const showMapper = ref(false)
const maxRowsLimit = 100

const chartCanvas = ref(null)
let myChart = null

// 2. FUNGSI LUKIS GRAF MULTI-DATASET
const renderChart = () => {
  if (myChart) myChart.destroy()
  
  let actualType = chartType.value
  if (['horizontalBar', 'stackedBar'].includes(actualType)) actualType = 'bar'
  if (actualType === 'area') actualType = 'line'
  if (actualType === 'mixed') actualType = 'bar' // Base canvas type untuk mixed chart mesti 'bar'

  const isPie = ['pie', 'doughnut', 'polarArea'].includes(actualType)

  const chartDatasets = datasetMeta.value.map((meta, index) => {
    let bgColors, borderColors;
    if (isPie && datasetMeta.value.length === 1) {
      const totalData = dataRows.value.length
      bgColors = dataRows.value.map((_, i) => {
        const step = totalData > 1 ? (200 / (totalData - 1)) : 0
        const alpha = Math.max(50, Math.round(255 - (i * step))).toString(16).padStart(2, '0')
        return meta.color + alpha
      })
      borderColors = '#ffffff'
    } else {
      bgColors = meta.color + 'cc'
      borderColors = meta.color
    }

    // LOGIK MIXED CHART: Dataset ke-2 (index 1) akan ditukar jadi 'line' secara automatik
    let datasetType = undefined
    let isFilled = chartType.value === 'area'
    
    if (chartType.value === 'mixed' && index === 1) {
      datasetType = 'line'
      bgColors = meta.color // Line tak perlu opacity pekat sangat
    }

    return {
      type: datasetType, // Chart.js benarkan override jenis per dataset
      label: meta.name,
      data: dataRows.value.map(row => row.values[meta.id] || 0),
      backgroundColor: bgColors,
      borderColor: borderColors,
      borderWidth: 2,
      fill: isFilled,
      tension: datasetType === 'line' ? 0.3 : 0 // Buat garisan lengkung sikit kalau line
    }
  })

  let specificOptions = { responsive: true, maintainAspectRatio: false }
  
  if (chartType.value === 'horizontalBar') {
    specificOptions.indexAxis = 'y'
  } else if (chartType.value === 'stackedBar') {
    specificOptions.scales = { x: { stacked: true }, y: { stacked: true } }
  }

  specificOptions.plugins = {
    title: { display: true, text: chartTitle.value, font: { size: 18, weight: 'bold' }, padding: { top: 10, bottom: 20 } }
  }

  myChart = new Chart(chartCanvas.value, {
    type: actualType,
    data: {
      labels: dataRows.value.map(row => row.label),
      datasets: chartDatasets
    },
    options: specificOptions
  })
}

// 3. FUNGSI CRUD DATASET & BARIS
const addDataset = () => {
  const newId = `ds${nextDsId++}`
  const color = colorPalette[(nextDsId - 4) % colorPalette.length]
  datasetMeta.value.push({ id: newId, name: `Set ${datasetMeta.value.length + 1}`, color })
  
  // Set nilai default 0 untuk semua baris sedia ada pada lajur baru ini
  dataRows.value.forEach(row => { row.values[newId] = 0 })
}

const removeDataset = (id) => {
  if (datasetMeta.value.length > 1) {
    datasetMeta.value = datasetMeta.value.filter(ds => ds.id !== id)
  } else {
    alert("Kena tinggal sekurang-kurangnya 1 set data bro!")
  }
}

const addRow = () => {
  const newVals = {}
  datasetMeta.value.forEach(ds => { newVals[ds.id] = 0 })
  dataRows.value.push({ id: nextRowId++, label: `Data ${dataRows.value.length + 1}`, values: newVals })
}

const removeRow = (id) => {
  if (dataRows.value.length > 1) {
    dataRows.value = dataRows.value.filter(row => row.id !== id)
  }
}

// 4. LIFECYCLE & WATCHER
onMounted(() => renderChart())
watch([chartTitle, chartType, dataRows, datasetMeta], () => renderChart(), { deep: true })

// 5. EXPORT PNG
const downloadPNG = () => {
  const link = document.createElement('a')
  link.href = chartCanvas.value.toDataURL('image/png')
  link.download = `${chartTitle.value.replace(/\s+/g, '-').toLowerCase()}.png`
  link.click()
}

// 6. COPY HTML SNIPPET (Diubah untuk sokong Multi-Dataset)
const isCopied = ref(false)
const snippetMode = ref('static') // Pilihan: 'static' atau 'dynamic'
const copyHTML = async () => {
  const isDynamic = snippetMode.value === 'dynamic'
  const labelsString = JSON.stringify(dataRows.value.map(row => row.label))
  
  let dynamicVariables = ''
  let datasetsJS = ''

  if (isDynamic) {
    // Mode Developer: Asingkan pembolehubah data di atas supaya mudah diganti
    dynamicVariables = `\n  // -- DEVELOPER: Masukkan data dari database/API anda di sini --\n  const dbLabels = ${labelsString};\n`
    
    datasetsJS = datasetMeta.value.map((meta, index) => {
      const dataArr = isDynamic ? `dbData_${index + 1}` : JSON.stringify(dataRows.value.map(row => row.values[meta.id] || 0))
      
      // Override type untuk mixed chart dalam kod salinan
      let overrideType = ''
      let lineTension = ''
      if (chartType.value === 'mixed' && index === 1) {
        overrideType = "\n        type: 'line',"
        lineTension = "\n        tension: 0.3,"
      }
      
      return `{${overrideType}
        label: '${meta.name}',
        data: ${dataArr},
        backgroundColor: '${meta.color}cc',
        borderColor: '${meta.color}',
        borderWidth: 2,
        fill: ${chartType.value === 'area'},${lineTension}
      }`
    }).join(',\n        ')
    
    dynamicVariables += `\n` // Jarakkan sikit supaya kemas
  } else {
    // Mode Biasa: Data di-hardcode terus dalam konfigurasi Chart.js
    datasetsJS = datasetMeta.value.map(meta => {
      const dataArr = JSON.stringify(dataRows.value.map(row => row.values[meta.id] || 0))
      return `{
        label: '${meta.name}',
        data: ${dataArr},
        backgroundColor: '${meta.color}cc',
        borderColor: '${meta.color}',
        borderWidth: 2
      }`
    }).join(',\n        ')
  }

  const labelInjection = isDynamic ? 'dbLabels' : labelsString

  let actualType = chartType.value
  if (['horizontalBar', 'stackedBar'].includes(actualType)) actualType = 'bar'
  if (actualType === 'area') actualType = 'line'
  if (actualType === 'mixed') actualType = 'bar'

  let extraOptions = ''
  if (chartType.value === 'horizontalBar') extraOptions = `\n      indexAxis: 'y',`
  else if (chartType.value === 'stackedBar') extraOptions = `\n      scales: { x: { stacked: true }, y: { stacked: true } },`

  const htmlSnippet = `
<div style="width: 100%; max-width: 800px; margin: auto;">
  <canvas id="grafKomuniti"></canvas>
</div>
<script src="https://cdn.jsdelivr.net/npm/chart.js"><\/script>
<script>${dynamicVariables}
  const ctx = document.getElementById('grafKomuniti');
  new Chart(ctx, {
    type: '${actualType}',
    data: {
      labels: ${labelInjection},
      datasets: [
        ${datasetMeta.value.map((meta, index) => {
          const dataArr = isDynamic ? `dbData_${index + 1}` : JSON.stringify(dataRows.value.map(row => row.values[meta.id] || 0));
          return `{
            label: '${meta.name}',
            data: ${dataArr},
            backgroundColor: '${meta.color}cc',
            borderColor: '${meta.color}',
            borderWidth: 2,
            fill: ${chartType.value === 'area'}
          }`
        }).join(',\n        ')}
      ]
    },
    options: { 
      responsive: true,${extraOptions}
      plugins: {
        title: { display: true, text: '${chartTitle.value}', font: { size: 18, weight: 'bold' }, padding: { top: 10, bottom: 20 } }
      }
    }
  });
<\/script>
  `.trim()

  try {
    await navigator.clipboard.writeText(htmlSnippet)
    isCopied.value = true
    setTimeout(() => { isCopied.value = false }, 2000)
  } catch (err) { alert("Gagal menyalin kod.") }
}

// 7. FUNGSI UPLOAD EXCEL MULTI-DATASET
const uploadData = async (event) => {
  const file = event.target.files[0]
  if (!file) return

  const reader = new FileReader()
  reader.onload = (e) => {
    const data = new Uint8Array(e.target.result)
    const workbook = XLSX.read(data, { type: 'array' })
    const worksheet = workbook.Sheets[workbook.SheetNames[0]]
    const parsedData = XLSX.utils.sheet_to_json(worksheet, { header: 1, blankrows: false })

    if (parsedData.length > 1) {
      excelHeaders.value = parsedData[0] || []
      rawExcelData.value = parsedData.slice(1)
      
      selectedLabelCol.value = excelHeaders.value[0] || ''
      selectedDataCols.value = [excelHeaders.value[1]] || [] // Default pilih 1 lajur data
      
      showMapper.value = true
    }
  }
  reader.readAsArrayBuffer(file)
  event.target.value = ''
}

// 8. APPLY MAPPING MULTI-DATASET
const applyMapping = () => {
  if (!selectedLabelCol.value || selectedDataCols.value.length === 0) return

  const labelIndex = excelHeaders.value.indexOf(selectedLabelCol.value)
  
  // Bina datasetMeta baharu berdasarkan lajur yang user tanda (checkbox)
  datasetMeta.value = selectedDataCols.value.map((colName, idx) => ({
    id: `excel_ds_${idx}`,
    name: String(colName),
    color: colorPalette[idx % colorPalette.length]
  }))

  dataRows.value = []
  nextRowId = 1
  let rowsProcessed = 0

  for (const row of rawExcelData.value) {
    if (rowsProcessed >= maxRowsLimit) break

    const rawLabel = row[labelIndex]
    if (rawLabel !== undefined) {
      const label = String(rawLabel).trim()
      if (label) {
        const rowValues = {}
        
        // Kutip data untuk setiap lajur yang dipilih
        datasetMeta.value.forEach((ds) => {
          const dataIndex = excelHeaders.value.indexOf(ds.name)
          const val = parseFloat(row[dataIndex])
          rowValues[ds.id] = isNaN(val) ? 0 : val
        })
        
        dataRows.value.push({ id: nextRowId++, label, values: rowValues })
        rowsProcessed++
      }
    }
  }
}
</script>

<template>
  <div class="min-h-screen bg-gray-50 p-8 font-sans text-gray-800">
    <div class="max-w-6xl mx-auto bg-white rounded-xl shadow-xl p-6 md:p-8">
      
      <header class="mb-8 border-b pb-4">
        <h1 class="text-3xl font-extrabold text-gray-900">From datas to chart</h1>
        <p class="text-gray-500 text-sm mt-1">Build, Export & Copy HTML Snippet</p>
      </header>

      <div class="grid grid-cols-1 lg:grid-cols-12 gap-8">
        
        <!-- BAHAGIAN KIRI: Form Input Data -->
        <div class="lg:col-span-5 bg-gray-50 p-6 rounded-xl border border-gray-100 shadow-sm flex flex-col">
          <h2 class="text-lg font-bold mb-4 border-b pb-2">1. Tetapan Data</h2>
          
          <div class="grid grid-cols-1 md:grid-cols-2 gap-4 mb-6">
            <div>
              <label class="block text-xs font-semibold text-gray-600 uppercase mb-1">Tajuk</label>
              <input v-model="chartTitle" type="text" class="w-full p-2 border rounded shadow-sm focus:ring-2 focus:ring-blue-500 outline-none">
            </div>
            <div>
              <label class="block text-xs font-semibold text-gray-600 uppercase mb-1">Jenis</label>
              <select v-model="chartType" class="w-full p-2 border rounded shadow-sm focus:ring-2 focus:ring-blue-500 outline-none bg-white text-sm">
                <optgroup label="📈 Trend & Masa (Bulan/Tahun)">
                  <option value="line">Line (Garis)</option>
                  <option value="area">Area (Kawasan)</option>
                </optgroup>
                <optgroup label="📊 Perbandingan (Kategori/Produk)">
                  <option value="bar">Bar (Tegak)</option>
                  <option value="horizontalBar">Bar (Mendatar)</option>
                  <option value="stackedBar">Bar (Bertindih)</option>
                  <option value="radar">Radar</option>
                </optgroup>
                <optgroup label="📈 Trend & Perbandingan Lanjutan">
                  <option value="line">Line (Garis)</option>
                  <option value="area">Area (Kawasan)</option>
                  <option value="mixed">Mixed (Bar + Line)</option>
                </optgroup>
                <optgroup label="🍩 Komposisi (Mesti 1 Set Data Sahaja)">
                  <option value="pie">Pie</option>
                  <option value="doughnut">Doughnut</option>
                  <option value="polarArea">Polar Area</option>
                </optgroup>
              </select>
            </div>
          </div>

          <!-- UPLOAD & MAPPER EXCEL -->
          <div class="mb-4 p-4 bg-indigo-50 border border-indigo-100 rounded-lg shadow-sm">
            <div class="flex items-center justify-between mb-2">
              <div>
                <p class="text-sm font-bold text-indigo-900">Ada data yang banyak?</p>
                <p class="text-xs text-indigo-700 mt-0.5">Upload CSV atau Excel</p>
              </div>
              <label class="cursor-pointer bg-indigo-600 hover:bg-indigo-700 text-white text-xs font-bold py-2 px-3 rounded shadow transition-all active:scale-95">
                📁 Upload
                <input type="file" accept=".csv, .xlsx, .xls" class="hidden" @change="uploadData">
              </label>
            </div>

            <!-- UI Mapper Baharu (Checkboxes) -->
            <div v-if="showMapper" class="mt-4 pt-4 border-t border-indigo-200 animate-pulse-once">
              <h3 class="text-sm font-bold text-indigo-900 mb-2">⚙️ Susunan Lajur</h3>
              
              <div class="mb-3">
                <label class="block text-xs font-semibold text-indigo-700 mb-1">Paksi-X (Label)</label>
                <select v-model="selectedLabelCol" class="w-full p-1 border rounded text-sm bg-white outline-none">
                  <option v-for="h in excelHeaders" :key="'l-'+h" :value="h">{{ h }}</option>
                </select>
              </div>
              
              <div class="mb-4">
                <label class="block text-xs font-semibold text-indigo-700 mb-1">Paksi-Y (Tanda lajur data yang mahu dipaparkan)</label>
                <div class="grid grid-cols-2 gap-2 bg-white p-2 border rounded max-h-32 overflow-y-auto">
                  <label v-for="h in excelHeaders" :key="'d-'+h" class="flex items-center gap-2 text-xs cursor-pointer">
                    <input type="checkbox" :value="h" v-model="selectedDataCols" class="cursor-pointer">
                    {{ h }}
                  </label>
                </div>
              </div>
              
              <button @click="applyMapping" class="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-2 rounded shadow transition-colors text-sm">
                Jana Graf
              </button>
            </div>
          </div>

          <!-- GRID DATA DINAMIK (Boleh scroll melintang) -->
          <div class="flex-1 min-h-[300px] overflow-hidden flex flex-col border rounded shadow-inner bg-white">
            <div class="overflow-x-auto p-3">
              
              <!-- Header Lajur (Senarai Dataset) -->
              <div class="flex gap-2 items-center mb-2 pb-2 border-b min-w-max">
                <div class="w-28 flex-shrink-0 font-bold text-[10px] uppercase text-gray-500 pl-6">Label X</div>
                <div v-for="ds in datasetMeta" :key="ds.id" class="flex-shrink-0 w-32 bg-gray-100 rounded border p-1 flex items-center gap-1 shadow-sm">
                  <input type="color" v-model="ds.color" class="w-5 h-5 p-0 border-0 cursor-pointer rounded" title="Tukar Warna">
                  <input type="text" v-model="ds.name" class="flex-1 w-full text-xs font-bold text-gray-700 bg-transparent outline-none">
                  <button @click="removeDataset(ds.id)" class="text-red-400 hover:text-red-600 font-bold px-1">&times;</button>
                </div>
                <button @click="addDataset" class="text-xs bg-blue-50 text-blue-600 px-2 py-1.5 rounded border border-blue-200 hover:bg-blue-100 font-semibold shadow-sm">+ Set Data</button>
              </div>

              <!-- Baris Nilai Data -->
              <div class="space-y-2 min-w-max">
                <div v-for="(row, index) in dataRows" :key="row.id" class="flex gap-2 items-center">
                  <span class="text-gray-400 font-bold text-xs w-4 flex-shrink-0 text-right">{{ index + 1 }}.</span>
                  <input v-model="row.label" type="text" class="w-28 flex-shrink-0 p-1.5 border-b focus:border-blue-500 outline-none text-sm bg-gray-50 rounded-t">
                  
                  <input v-for="ds in datasetMeta" :key="ds.id" v-model.number="row.values[ds.id]" type="number" class="w-32 flex-shrink-0 p-1.5 border-b focus:border-blue-500 outline-none text-sm text-center">
                  
                  <button @click="removeRow(row.id)" class="text-red-300 hover:text-red-500 p-1 font-bold text-lg">&times;</button>
                </div>
              </div>
              
            </div>
            
            <div class="p-3 bg-gray-50 border-t mt-auto">
              <button @click="addRow" class="w-full bg-white text-gray-700 font-semibold py-1.5 rounded border shadow-sm hover:bg-gray-100 transition-colors text-sm">
                + Tambah Baris
              </button>
            </div>
          </div>
        </div>

        <!-- BAHAGIAN KANAN: Preview & Actions -->
        <div class="lg:col-span-7 flex flex-col">
          <h2 class="text-lg font-bold mb-4 border-b pb-2">2. Preview & Export</h2>
          
          <div class="flex-1 bg-white p-4 border rounded-xl shadow-inner mb-6 min-h-[400px] relative flex justify-center items-center">
            <div class="w-full h-full relative" style="height: 400px;">
              <canvas ref="chartCanvas"></canvas>
            </div>
          </div>

          <div class="flex flex-col gap-2">
            <div class="flex flex-col sm:flex-row gap-4">
              <!-- Butang Export -->
              <button @click="downloadPNG" class="flex-1 bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-3 px-3 rounded-lg shadow-md transition-all active:scale-95">
                📥 Export PNG
              </button>
              
              <!-- Bahagian Copy HTML dengan Dropdown -->
              <div class="flex-1 flex gap-2">
                <select v-model="snippetMode" class="w-1/3 bg-gray-100 border border-gray-300 text-gray-700 font-semibold py-3 px-2 rounded-lg shadow-sm outline-none focus:ring-2 focus:ring-gray-400 text-sm cursor-pointer" title="Pilih Format Kod">
                  <option value="static">Data Statik</option>
                  <option value="dynamic">Mod Dev</option>
                </select>
                
                <button @click="copyHTML" 
                  :class="isCopied ? 'bg-green-600 hover:bg-green-700' : 'bg-gray-800 hover:bg-gray-900'"
                  class="flex-1 text-white font-bold py-3 px-4 rounded-lg shadow-md transition-all active:scale-95 flex items-center justify-center gap-2 text-sm whitespace-nowrap">
                  <span v-if="!isCopied">📋 Copy HTML</span>
                  <span v-else>✅ Kod Disalin!</span>
                </button>
              </div>
            </div>

            <!-- Nota Panduan Pintar (Ikut Mode) -->
            <p class="text-xs text-gray-500 px-1">
              <span v-if="snippetMode === 'dynamic'">
                💡 <span class="font-semibold text-indigo-600">Mod Dev Aktif:</span> Mengasingkan pembolehubah <code class="bg-gray-100 px-1 py-0.5 rounded text-gray-700 font-mono">dbLabels</code> & <code class="bg-gray-100 px-1 py-0.5 rounded text-gray-700 font-mono">dbData</code> untuk sambung terus ke database/backend.
              </span>
              <span v-else>
                💡 <span class="font-semibold text-gray-600">Mod Statik:</span> Data nilai di-hardcode terus ke dalam struktur tetapan Chart.js.
              </span>
            </p>
          </div>
        </div>

      </div>
    </div>
  </div>
</template>