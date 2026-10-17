<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'

type Row = { key: string; label: string; value: string }

const now = ref(new Date())
const rows = ref<Row[]>([])
const note = ref('')
const savedAt = ref('')
const copied = ref(false)
const storageOk = ref(true)

let timer: number | undefined

const NOTE_KEY = 'probe.note'
const NOTE_AT = 'probe.note.at'

function fmt(d: Date): string {
  const p = (n: number) => String(n).padStart(2, '0')
  return `${d.getFullYear()}-${p(d.getMonth() + 1)}-${p(d.getDate())} ${p(d.getHours())}:${p(d.getMinutes())}:${p(d.getSeconds())}`
}


/** 从 UA 里解析出可读的名字与版本，避免直接甩一整串 UA */
function describeUa(ua: string): { name: string; version: string; engine: string } {
  const rules: Array<[RegExp, string, string]> = [
    [/\bEdg\/([\d.]+)/, 'Microsoft Edge', 'Blink'],
    [/\bOPR\/([\d.]+)/, 'Opera', 'Blink'],
    [/\bFirefox\/([\d.]+)/, 'Firefox', 'Gecko'],
    [/(?:Headless)?Chrom(?:e|ium)\/([\d.]+)/, 'Chrome', 'Blink'],
    [/\bVersion\/([\d.]+).*Safari/, 'Safari', 'WebKit'],
    [/\bSafari\/([\d.]+)/, 'Safari', 'WebKit'],
  ]
  for (const [re, name, engine] of rules) {
    const m = ua.match(re)
    if (m) return { name, version: m[1].split('.').slice(0, 3).join('.'), engine }
  }
  return { name: '未知', version: '-', engine: '未知' }
}

function detectPlatform(ua: string): string {
  if (/iPhone|iPad|iPod/.test(ua)) return 'iOS'
  if (/Android/.test(ua)) return 'Android'
  if (/Macintosh|Mac OS X/.test(ua)) return 'macOS'
  if (/Windows/.test(ua)) return 'Windows'
  if (/Linux/.test(ua)) return 'Linux'
  return '未知'
}

function collect(): Row[] {
  const ua = navigator.userAgent
  const d = describeUa(ua)
  const nav = navigator as Navigator & { connection?: { effectiveType?: string; downlink?: number } }
  const conn = nav.connection

  const out: Row[] = [
    { key: 'origin', label: '访问域名', value: location.origin },
    { key: 'platform', label: '客户端平台', value: detectPlatform(ua) },
    { key: 'browser', label: '浏览器', value: `${d.name} ${d.version}` },
    { key: 'engine', label: '渲染引擎', value: d.engine },
    { key: 'viewport', label: '视口尺寸', value: `${window.innerWidth} × ${window.innerHeight}` },
    { key: 'screen', label: '屏幕分辨率', value: `${screen.width} × ${screen.height}` },
    { key: 'dpr', label: '像素比', value: String(window.devicePixelRatio || 1) },
    { key: 'lang', label: '语言', value: navigator.language },
    {
      key: 'tz',
      label: '时区',
      value: Intl.DateTimeFormat().resolvedOptions().timeZone || '-',
    },
    { key: 'online', label: '网络状态', value: navigator.onLine ? '在线' : '离线' },
    {
      key: 'conn',
      label: '连接类型',
      value: conn?.effectiveType ? `${conn.effectiveType} / ${conn.downlink ?? '-'} Mbps` : '浏览器未提供',
    },
    { key: 'storage', label: '本地存储', value: storageOk.value ? '可用' : '不可用' },
    { key: 'langs', label: '语言偏好', value: (navigator.languages || [navigator.language]).join(', ') },
  ]
  return out
}

function refresh() {
  rows.value = collect()
}

function saveNote() {
  try {
    localStorage.setItem(NOTE_KEY, note.value)
    localStorage.setItem(NOTE_AT, new Date().toISOString())
    savedAt.value = fmt(new Date())
    storageOk.value = true
  } catch {
    storageOk.value = false
  }
}

function clearNote() {
  note.value = ''
  savedAt.value = ''
  try {
    localStorage.removeItem(NOTE_KEY)
    localStorage.removeItem(NOTE_AT)
  } catch {
    storageOk.value = false
  }
}

async function copyAll() {
  const payload = {
    note: note.value,
    savedAt: savedAt.value,
    collectedAt: fmt(now.value),
    probe: Object.fromEntries(rows.value.map((r) => [r.key, r.value])),
    userAgent: navigator.userAgent,
  }
  const text = JSON.stringify(payload, null, 2)
  try {
    await navigator.clipboard.writeText(text)
  } catch {
    const ta = document.createElement('textarea')
    ta.value = text
    document.body.appendChild(ta)
    ta.select()
    document.execCommand('copy')
    ta.remove()
  }
  copied.value = true
  window.setTimeout(() => (copied.value = false), 1600)
}

watch(note, () => {
  if (storageOk.value) return
  storageOk.value = true
})

onMounted(() => {
  try {
    note.value = localStorage.getItem(NOTE_KEY) ?? ''
    const at = localStorage.getItem(NOTE_AT)
    savedAt.value = at ? fmt(new Date(at)) : ''
  } catch {
    storageOk.value = false
  }
  refresh()
  window.addEventListener('resize', refresh)
  window.addEventListener('online', refresh)
  window.addEventListener('offline', refresh)
  timer = window.setInterval(() => (now.value = new Date()), 1000)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', refresh)
  window.removeEventListener('online', refresh)
  window.removeEventListener('offline', refresh)
  if (timer) window.clearInterval(timer)
})

const clock = computed(() => fmt(now.value))
</script>

<template>
  <main class="page">
    <header class="head">
      <div class="brand">
        <span class="dot" aria-hidden="true"></span>
        <h1>环境探针</h1>
      </div>
      <p class="sub">读取当前客户端与网关的真实运行信息，用于确认应用在懒猫微服上的部署状态。</p>
      <p class="clock">当前时间 <strong>{{ clock }}</strong></p>
    </header>

    <section class="card">
      <div class="card-head">
        <h2>运行环境</h2>
        <button class="btn ghost" type="button" @click="refresh">重新读取</button>
      </div>
      <dl class="rows">
        <div v-for="r in rows" :key="r.key" class="row">
          <dt>{{ r.label }}</dt>
          <dd>{{ r.value }}</dd>
        </div>
      </dl>
    </section>

    <section class="card">
      <div class="card-head">
        <h2>本地便签</h2>
        <span class="hint">{{ savedAt ? '上次保存 ' + savedAt : '尚未保存' }}</span>
      </div>
      <p class="tip">内容保存在浏览器本地，刷新页面或重启应用后仍在，可用于验证数据持久化。</p>
      <textarea
        v-model="note"
        class="note"
        rows="4"
        maxlength="2000"
        placeholder="在这里写点什么，然后点保存…"
      ></textarea>
      <div class="actions">
        <button class="btn primary" type="button" @click="saveNote">保存</button>
        <button class="btn" type="button" :disabled="!note" @click="clearNote">清空</button>
        <button class="btn" type="button" @click="copyAll">{{ copied ? '已复制' : '复制全部信息' }}</button>
      </div>
    </section>

    <footer class="foot">
      <span>环境探针 · 懒猫微服应用</span>
      <span>{{ rows.length }} 项运行信息</span>
    </footer>
  </main>
</template>
