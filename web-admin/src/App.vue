<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue'
import {
  NButton,
  NCheckbox,
  NConfigProvider,
  NEmpty,
  NForm,
  NFormItem,
  NIcon,
  NInput,
  NMessageProvider,
  NSelect,
  NSpace,
  NSpin,
  NStatistic,
  NSwitch,
  NTag,
  createDiscreteApi
} from 'naive-ui'
import {
  AnalyticsOutline,
  ClipboardOutline,
  CopyOutline,
  DocumentTextOutline,
  FlashOutline,
  GlobeOutline,
  LogOutOutline,
  ChatbubbleEllipsesOutline,
  PlayOutline,
  RefreshOutline,
  SaveOutline,
  SettingsOutline,
  ShieldCheckmarkOutline,
  TerminalOutline,
  TrashOutline
} from '@vicons/ionicons5'

const { message } = createDiscreteApi(['message'])

const themeOverrides = {
  common: {
    primaryColor: '#18a058',
    primaryColorHover: '#36ad6a',
    primaryColorPressed: '#0c7a43',
    primaryColorSuppl: '#36ad6a',
    infoColor: '#2080f0',
    infoColorHover: '#4098fc',
    infoColorPressed: '#1060c9',
    borderRadius: '8px',
    fontFamily: 'Lato, "Microsoft YaHei UI", "Segoe UI", Arial, sans-serif',
    fontFamilyMono: '"Fira Code", Consolas, monospace'
  },
  Button: {
    borderRadiusMedium: '8px',
    fontWeight: '700'
  }
}

const navItems = [
  { key: 'overview', label: 'Prezentare generală', icon: AnalyticsOutline },
  { key: 'chat', label: 'Conversație', icon: ChatbubbleEllipsesOutline },
  { key: 'network', label: 'Rețea', icon: GlobeOutline },
  { key: 'test', label: 'Testare Serviciu', icon: FlashOutline },
  { key: 'settings', label: 'Configurări', icon: SettingsOutline },
  { key: 'logs', label: 'Jurnale', icon: TerminalOutline }
]

const endpointOptions = [
  { label: 'OpenAI Chat Completions', value: 'chat' },
  { label: 'OpenAI Responses', value: 'responses' },
  { label: 'Google generateContent', value: 'google' },
  { label: 'Google streamGenerateContent', value: 'google-stream' }
]

const active = ref('overview')
const loading = ref(false)
const saving = ref(false)
const testing = ref(false)
const networkLoading = ref(false)
const autoLogs = ref(true)
const stickToBottom = ref(true)
const logBox = ref(null)
const logTimer = ref(null)
const apiKeysText = ref('')
const apiKeyItems = ref([])
const apiKeyDraft = ref('')
const editingApiKeyIndex = ref(null)
const cookieItems = ref([])
const cookieDraft = ref('')
const editingCookieIndex = ref(null)

const auth = reactive({
  checked: false,
  authenticated: false,
  password: '',
  loggingIn: false
})

const status = reactive({
  ok: false,
  version: '',
  models: [],
  config: {},
  urls: {},
  logs: {},
  admin_static: {}
})

const config = reactive({
  cookie_file: '',
  cookie_files: [],
  cookie_content: '',
  cookie_contents: [],
  cookie_source: {},
  proxy: '',
  default_model: '',
  public_base_url: '',
  empty_response_fallback: '',
  api_keys: [],
  force_non_stream: false,
  admin_password: ''
})

const network = reactive({
  local_ip: '',
  public_ip: '',
  city: '',
  region: '',
  country: '',
  org: '',
  timezone: '',
  proxy_enabled: false,
  connectivity: {},
  raw_error: ''
})

const test = reactive({
  endpoint: 'chat',
  model: 'gemini-3.5-flash',
  stream: false,
  prompt: 'Bună, te rog să explici într-o propoziție dacă serviciul curent este funcțional.',
  result: ''
})

const chat = reactive({
  model: 'gemini-3.5-flash',
  stream: false,
  input: '',
  messages: []
})

const logs = reactive({
  text: '',
  offset: null,
  size: 0
})

const pageMeta = computed(() => ({
  overview: ['Prezentare generală', 'Vizualizați starea serviciului, adresele de apel și mediul de rulare'],
  chat: ['Conversație', 'Selectați un model și conversați direct cu serviciul curent'],
  network: ['Rețea', 'Vizualizați IP-ul public, locația și testați conectivitatea'],
  test: ['Testare Serviciu', 'Inițiați un apel compatibil cu OpenAI sau Gemini'],
  settings: ['Configurări', 'Ajustați Cookie, proxy, chei, parola de administrator și adresa publică'],
  logs: ['Jurnale', 'Vizualizați în timp real ieșirile serviciului și ale managerului desktop']
}[active.value]))

const modelOptions = computed(() => [
  { label: 'Toate modelele (folosește modelul implicit)', value: '__all__' },
  ...status.models.map((item) => ({
    label: `${item.id} - ${item.description || 'model'}`,
    value: item.id
  }))
])

const currentEndpoint = computed(() => endpointOptions.find((item) => item.value === test.endpoint))
const healthyType = computed(() => status.ok ? 'success' : 'error')
const cookieState = computed(() => config.cookie_file ? 'Configurat' : 'Mod anonim')
const proxyState = computed(() => config.proxy ? config.proxy : 'Mediu de sistem')
const locationText = computed(() => [network.country, network.region, network.city].filter(Boolean).join(' / ') || 'Neobținut')

function selectedModel(value) {
  return value && value !== '__all__' ? value : (config.default_model || status.models[0]?.id || 'gemini-3.5-flash')
}

const curlCommand = computed(() => {
  const model = selectedModel(test.model)
  const prompt = test.prompt.replace(/"/g, '\\"')
  if (test.endpoint === 'responses') {
    return `curl ${status.urls.current || '/v1'}/responses -H "Content-Type: application/json" -d "{\"model\":\"${model}\",\"input\":\"${prompt}\"}"`
  }
  if (test.endpoint.startsWith('google')) {
    const method = test.endpoint === 'google-stream' ? 'streamGenerateContent' : 'generateContent'
    return `curl ${window.location.origin}/v1beta/models/${model}:${method} -H "Content-Type: application/json" -d "{\"contents\":[{\"role\":\"user\",\"parts\":[{\"text\":\"${prompt}\"}]}]}"`
  }
  return `curl ${status.urls.current || '/v1'}/chat/completions -H "Content-Type: application/json" -d "{\"model\":\"${model}\",\"messages\":[{\"role\":\"user\",\"content\":\"${prompt}\"}],\"stream\":${test.stream}}"`
})

function pretty(data) {
  if (typeof data === 'string') {
    try { return JSON.stringify(JSON.parse(data), null, 2) } catch (_) { return data }
  }
  return JSON.stringify(data, null, 2)
}

async function api(path, options = {}) {
  const res = await fetch(path, options)
  const text = await res.text()
  let data = text
  try { data = text ? JSON.parse(text) : {} } catch (_) {}
  if (!res.ok) {
    if (res.status === 401 && path !== '/admin/api/login') auth.authenticated = false
    const reason = data?.error?.message || data?.error || res.statusText
    throw new Error(reason)
  }
  return data
}

function applyStatus(data) {
  Object.assign(status, data)
  Object.assign(config, data.config || {})
  config.admin_password = ''
  apiKeyItems.value = [...(data.config?.api_keys || [])]
  apiKeysText.value = apiKeyItems.value.join('\n')
  cookieItems.value = (data.config?.cookie_files || []).map((path, index) => ({ path, content: '', label: `Cookie ${index + 1}` }))
  if (!test.model && status.models.length) test.model = status.models[0].id
}

function syncApiKeysText() {
  apiKeysText.value = apiKeyItems.value.join('\n')
}

function maskSecret(value) {
  if (!value) return '-'
  if (value.length <= 10) return `${value.slice(0, 3)}***${value.slice(-2)}`
  return `${value.slice(0, 7)}...${value.slice(-4)}`
}

function addApiKey() {
  const value = apiKeyDraft.value.trim()
  if (!value) return
  if (editingApiKeyIndex.value !== null) {
    apiKeyItems.value.splice(editingApiKeyIndex.value, 1, value)
    editingApiKeyIndex.value = null
  } else {
    apiKeyItems.value.push(value)
  }
  apiKeyDraft.value = ''
  syncApiKeysText()
}

function editApiKey(index) {
  editingApiKeyIndex.value = index
  apiKeyDraft.value = apiKeyItems.value[index]
}

function deleteApiKey(index) {
  apiKeyItems.value.splice(index, 1)
  if (editingApiKeyIndex.value === index) {
    editingApiKeyIndex.value = null
    apiKeyDraft.value = ''
  }
  syncApiKeysText()
}

function copyApiKey(index) {
  copyText(apiKeyItems.value[index], 'Cheie API copiată')
}

function addCookie() {
  const value = cookieDraft.value.trim()
  if (!value) return
  const row = { path: '', content: value, label: `Cookie ${editingCookieIndex.value === null ? cookieItems.value.length + 1 : editingCookieIndex.value + 1}` }
  if (editingCookieIndex.value !== null) {
    cookieItems.value.splice(editingCookieIndex.value, 1, row)
    editingCookieIndex.value = null
  } else {
    cookieItems.value.push(row)
  }
  cookieDraft.value = ''
}

function editCookie(index) {
  editingCookieIndex.value = index
  cookieDraft.value = cookieItems.value[index].content || ''
}

function deleteCookie(index) {
  cookieItems.value.splice(index, 1)
  if (editingCookieIndex.value === index) {
    editingCookieIndex.value = null
    cookieDraft.value = ''
  }
}

async function checkAuth() {
  try {
    const data = await api('/admin/api/auth')
    auth.authenticated = !!data.authenticated
    if (auth.authenticated) await bootDashboard()
  } catch (_) {
    auth.authenticated = false
  } finally {
    auth.checked = true
  }
}

async function login() {
  if (!auth.password) {
    message.warning('Vă rugăm introduceți parola de administrator')
    return
  }
  auth.loggingIn = true
  try {
    await api('/admin/api/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ password: auth.password })
    })
    auth.authenticated = true
    auth.password = ''
    await bootDashboard()
    message.success('Autentificare reușită')
  } catch (err) {
    message.error(`Autentificare eșuată: ${err.message}`)
  } finally {
    auth.loggingIn = false
  }
}

async function logout() {
  await api('/admin/api/logout', { method: 'POST' }).catch(() => {})
  auth.authenticated = false
  if (logTimer.value) clearInterval(logTimer.value)
  logTimer.value = null
}

async function bootDashboard() {
  await loadStatus()
  await readLogs(true)
  toggleLogTimer()
}

async function loadStatus(showToast = false) {
  loading.value = true
  try {
    applyStatus(await api('/admin/api/status'))
    if (showToast) message.success('Stare reîmprospătată')
  } catch (err) {
    status.ok = false
    message.error(`Citire stare eșuată: ${err.message}`)
  } finally {
    loading.value = false
  }
}

async function loadNetwork(showToast = false) {
  networkLoading.value = true
  try {
    Object.assign(network, await api('/admin/api/network'))
    if (showToast) message.success('Informații rețea reîmprospătate')
  } catch (err) {
    message.error(`Detectare rețea eșuată: ${err.message}`)
  } finally {
    networkLoading.value = false
  }
}

async function saveConfig() {
  saving.value = true
  try {
    syncApiKeysText()
    const cookieContents = cookieItems.value.map((item) => item.content).filter(Boolean)
    const cookieFiles = cookieItems.value.filter((item) => !item.content && item.path).map((item) => item.path)
    const payload = {
      ...config,
      api_keys: [...apiKeyItems.value],
      cookie_files: cookieContents.length ? undefined : cookieFiles,
      cookie_contents: cookieContents.length ? cookieContents : undefined,
      cookie_content: ''
    }
    const data = await api('/admin/api/config', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    })
    Object.assign(config, data.config || {})
    config.admin_password = ''
    apiKeyItems.value = [...(data.config?.api_keys || [])]
    apiKeysText.value = apiKeyItems.value.join('\n')
    cookieItems.value = (data.config?.cookie_files || []).map((path, index) => ({ path, content: '', label: `Cookie ${index + 1}` }))
    await loadStatus()
    message.success('Configurări salvate')
  } catch (err) {
    message.error(`Salvare eșuată: ${err.message}`)
  } finally {
    saving.value = false
  }
}

function requestForTest() {
  const model = selectedModel(test.model)
  if (test.endpoint === 'responses') {
    return ['/v1/responses', { model, input: test.prompt, stream: false }]
  }
  if (test.endpoint === 'google' || test.endpoint === 'google-stream') {
    const method = test.endpoint === 'google-stream' ? 'streamGenerateContent' : 'generateContent'
    return [`/v1beta/models/${encodeURIComponent(model)}:${method}`, {
      contents: [{ role: 'user', parts: [{ text: test.prompt }] }]
    }]
  }
  return ['/v1/chat/completions', {
    model,
    messages: [{ role: 'user', content: test.prompt }],
    stream: test.stream && !config.force_non_stream
  }]
}

const callGuide = computed(() => {
  const base = status.urls.current || '/v1'
  const model = selectedModel(test.model)
  const key = apiKeysText.value.split(/\n|,/).map((item) => item.trim()).filter(Boolean)[0] || 'sk-your-key'
  return [
    `URL de bază: ${base}`,
    `Cheie API: ${apiKeysText.value ? key : 'Poate fi completat arbitrar când cheile nu sunt activate'}`,
    `Chat: POST ${base}/chat/completions`,
    `Răspunsuri: POST ${base}/responses`,
    `Model: ${model}`,
    `Streaming: ${config.force_non_stream ? 'Dezactivat global, chiar dacă stream=true din exterior, va fi procesat ca non-streaming' : 'Controlat de parametrul stream al cererii'}`
  ].join('\n')
})

function parseChatResponse(text, streamed = false) {
  if (!streamed) {
    const data = JSON.parse(text)
    return data?.choices?.[0]?.message?.content || data?.choices?.[0]?.delta?.content || ''
  }
  let answer = ''
  for (const line of text.split(/\r?\n/)) {
    const trimmed = line.trim()
    if (!trimmed.startsWith('data:')) continue
    const payload = trimmed.slice(5).trim()
    if (!payload || payload === '[DONE]') continue
    try {
      const data = JSON.parse(payload)
      answer += data?.choices?.[0]?.delta?.content || data?.choices?.[0]?.message?.content || ''
    } catch (_) {}
  }
  return answer
}

async function runTest() {
  testing.value = true
  test.result = 'Se solicită...'
  try {
    const [path, body] = requestForTest()
    const res = await fetch(path, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body)
    })
    const text = await res.text()
    if (!res.ok) throw new Error(pretty(text))
    test.result = pretty(text)
  } catch (err) {
    test.result = `Cerere eșuată\n${err.message}`
    message.error('Test eșuat')
  } finally {
    testing.value = false
  }
}

async function sendChat() {
  const content = chat.input.trim()
  if (!content) {
    message.warning('Vă rugăm introduceți conținutul conversației')
    return
  }
  const model = selectedModel(chat.model)
  const messages = [...chat.messages, { role: 'user', content }]
  chat.messages = messages
  chat.input = ''
  testing.value = true
  try {
    const streamed = chat.stream && !config.force_non_stream
    const res = await fetch('/v1/chat/completions', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ model, messages, stream: streamed })
    })
    const text = await res.text()
    if (!res.ok) throw new Error(pretty(text))
    const answer = parseChatResponse(text, streamed)
    chat.messages.push({ role: 'assistant', content: answer || config.empty_response_fallback || 'Răspuns gol' })
  } catch (err) {
    chat.messages.push({ role: 'assistant', content: `Apel eșuat: ${err.message}` })
  } finally {
    testing.value = false
  }
}

async function readLogs(reset = false) {
  if (!auth.authenticated) return
  try {
    const query = reset || logs.offset === null ? '?tail=60000' : `?offset=${logs.offset}`
    const data = await api(`/admin/api/logs${query}`)
    logs.offset = data.offset
    logs.size = data.size
    logs.text = reset ? data.content : logs.text + data.content
    if (!data.exists && reset) logs.text = data.error ? `Jurnal indisponibil: ${data.error}` : 'Niciun fișier jurnal momentan, va fi afișat aici după ce serviciul generează ieșiri.'
    if (stickToBottom.value) await scrollLogs()
  } catch (err) {
    if (reset) logs.text = `Citire jurnal eșuată: ${err.message}`
    message.warning(`Citire jurnal eșuată: ${err.message}`)
  }
}

async function scrollLogs() {
  await nextTick()
  if (logBox.value) logBox.value.scrollTop = logBox.value.scrollHeight
}

function toggleLogTimer() {
  if (logTimer.value) clearInterval(logTimer.value)
  logTimer.value = autoLogs.value && auth.authenticated ? setInterval(() => readLogs(false), 1800) : null
}

async function copyText(text, label = 'Copiat') {
  await navigator.clipboard.writeText(text || '')
  message.success(label)
}

function openUrl(url) {
  if (url) window.open(url, '_blank', 'noopener,noreferrer')
}

watch(autoLogs, toggleLogTimer)
watch(active, (value) => {
  if (value === 'network' && !network.public_ip && !networkLoading.value) loadNetwork()
})

onMounted(checkAuth)

onBeforeUnmount(() => {
  if (logTimer.value) clearInterval(logTimer.value)
})
</script>

<template>
  <NConfigProvider :theme-overrides="themeOverrides">
    <NMessageProvider>
      <div v-if="!auth.checked" class="login-screen">
        <NSpin size="large" />
      </div>

      <div v-else-if="!auth.authenticated" class="login-screen">
        <section class="login-card">
          <div class="login-mark"><NIcon :component="ShieldCheckmarkOutline" /></div>
          <h1>GeminiWeb2API</h1>
          <p>După autentificarea în panoul de administrare, puteți vizualiza starea, modifica configurările, testa interfețele și citi jurnalele.</p>
          <NForm label-placement="top" @submit.prevent="login">
            <NFormItem label="Parolă administrator">
              <NInput v-model:value="auth.password" type="password" show-password-on="click" placeholder="implicit sk-admin" @keyup.enter="login" />
            </NFormItem>
          </NForm>
          <NButton type="primary" block size="large" :loading="auth.loggingIn" @click="login">Autentificare</NButton>
          <div class="login-note">La prima autentificare, parola implicită este sk-admin, o puteți modifica în pagina de configurări după autentificare.</div>
        </section>
      </div>

      <div v-else class="app-shell">
        <aside class="sidebar">
          <div class="brand">
            <div class="brand-mark">GW</div>
            <div>
              <div class="brand-title">GeminiWeb2API</div>
              <div class="brand-sub">Admin Console</div>
            </div>
          </div>
          <nav class="nav-list">
            <button v-for="item in navItems" :key="item.key" class="nav-button" :class="{ active: active === item.key }" @click="active = item.key">
              <NIcon size="18" :component="item.icon" />
              <span>{{ item.label }}</span>
            </button>
          </nav>
        </aside>

        <main class="main">
          <header class="topbar">
            <div>
              <h1 class="page-title">{{ pageMeta[0] }}</h1>
              <div class="page-desc">{{ pageMeta[1] }}</div>
            </div>
            <div class="top-actions">
              <NTag :type="healthyType" round>{{ status.ok ? 'Serviciu funcțional' : 'Eroare serviciu' }}</NTag>
              <NButton secondary :loading="loading" @click="loadStatus(true)"><template #icon><NIcon :component="RefreshOutline" /></template>Reîmprospătare</NButton>
              <NButton type="primary" secondary @click="active = 'chat'"><template #icon><NIcon :component="ChatbubbleEllipsesOutline" /></template>Conversație</NButton>
              <NButton secondary @click="logout"><template #icon><NIcon :component="LogOutOutline" /></template>Deconectare</NButton>
            </div>
          </header>

          <NSpin :show="loading && !status.version">
            <section v-show="active === 'overview'" class="content-grid">
              <div class="panel span-4 metric"><div class="metric-label">Versiune</div><div class="metric-value">{{ status.version || '-' }}</div><div class="metric-note">{{ status.admin_static?.ready ? 'Frontend construit' : 'Folosește pagina de avertizare lipsă' }}</div></div>
              <div class="panel span-4 metric"><div class="metric-label">Modele</div><div class="metric-value">{{ status.models.length }}</div><div class="metric-note">Implicit: {{ config.default_model || '-' }}</div></div>
              <div class="panel span-4 metric"><div class="metric-label">IP Public</div><div class="metric-value small-value">{{ network.public_ip || 'Neobținut' }}</div><div class="metric-note">{{ locationText }}</div></div>

              <div class="panel span-8">
                <div class="panel-head"><h2 class="panel-title">Adrese de apel</h2><NButton text type="primary" @click="copyText(JSON.stringify(status.urls, null, 2))"><template #icon><NIcon :component="CopyOutline" /></template>Copiază tot</NButton></div>
                <div class="url-list">
                  <div v-for="(value, key) in status.urls" :key="key" class="url-row">
                    <div class="url-label">{{ key }}</div><div class="url-value">{{ value || 'Neconfigurat' }}</div><NButton size="small" secondary @click="copyText(value)"><template #icon><NIcon :component="CopyOutline" /></template></NButton>
                  </div>
                </div>
              </div>

              <div class="panel span-4">
                <h2 class="panel-title">Mediu de rulare</h2>
                <NSpace vertical size="large">
                  <NStatistic label="Cookie" :value="cookieState" />
                  <NStatistic label="Proxy" :value="proxyState" />
                  <NStatistic label="Chei API" :value="apiKeysText ? 'Activat' : 'Dezactivat'" />
                  <NStatistic label="Director panou" :value="status.admin_static?.ready ? 'pregătit' : 'lipsește'" />
                </NSpace>
              </div>
            </section>

            <section v-show="active === 'chat'" class="content-grid chat-section">
              <div class="panel span-8 chat-panel">
                <div class="panel-head"><h2 class="panel-title">Conversație</h2><NSpace align="center" wrap><NTag type="info" round>{{ config.force_non_stream ? 'Non-streaming global' : (chat.stream ? 'Cerere streaming' : 'Cerere non-streaming') }}</NTag><NButton secondary @click="chat.messages = []"><template #icon><NIcon :component="TrashOutline" /></template>Golire</NButton></NSpace></div>
                <div class="chat-list">
                  <div v-if="!chat.messages.length" class="chat-empty">Selectați un model și introduceți conținutul pentru a iniția o conversație cu serviciul curent.</div>
                  <div v-for="(item, index) in chat.messages" :key="index" class="chat-message" :class="item.role">
                    <div class="chat-role">{{ item.role === 'user' ? 'Tu' : 'Asistent' }}</div>
                    <div class="chat-bubble">{{ item.content }}</div>
                  </div>
                </div>
                <NForm label-placement="top"><div class="form-grid">
                  <NFormItem label="Model"><NSelect v-model:value="chat.model" :options="modelOptions" filterable tag /></NFormItem>
                  <NFormItem label="Ieșire streaming"><NSwitch v-model:value="chat.stream" :disabled="config.force_non_stream" /></NFormItem>
                  <NFormItem label="Conținut" class="full"><NInput v-model:value="chat.input" type="textarea" placeholder="Introduceți conținutul conversației" :autosize="{ minRows: 4, maxRows: 10 }" @keydown.ctrl.enter.prevent="sendChat" /></NFormItem>
                </div></NForm>
                <div class="button-row"><NButton type="primary" :loading="testing" @click="sendChat"><template #icon><NIcon :component="PlayOutline" /></template>Trimite</NButton><NButton secondary @click="copyText(JSON.stringify(chat.messages, null, 2), 'Conversație copiată')"><template #icon><NIcon :component="CopyOutline" /></template>Copiază conversația</NButton></div>
              </div>
              <div class="panel span-4">
                <h2 class="panel-title">Instrucțiuni de apel</h2>
                <pre class="code-box compact-code">{{ callGuide }}</pre>
              </div>
            </section>

            <section v-show="active === 'network'" class="content-grid">
              <div class="panel span-12">
                <div class="panel-head"><h2 class="panel-title">Informații rețea</h2><NButton type="primary" :loading="networkLoading" @click="loadNetwork(true)"><template #icon><NIcon :component="RefreshOutline" /></template>Obține IP public / Testează conectivitatea</NButton></div>
                <div class="network-grid">
                  <div class="network-item"><span>IP rețea locală</span><strong>{{ network.local_ip || '-' }}</strong></div>
                  <div class="network-item"><span>IP Public</span><strong>{{ network.public_ip || '-' }}</strong></div>
                  <div class="network-item"><span>Locație</span><strong>{{ locationText }}</strong></div>
                  <div class="network-item"><span>Operator / ASN</span><strong>{{ network.org || '-' }}</strong></div>
                  <div class="network-item"><span>Fus orar</span><strong>{{ network.timezone || '-' }}</strong></div>
                  <div class="network-item"><span>Proxy</span><strong>{{ network.proxy_enabled ? 'Activat' : 'Dezactivat' }}</strong></div>
                </div>
              </div>
              <div class="panel span-6">
                <h2 class="panel-title">Conectivitate Gemini</h2>
                <pre class="code-box compact-code">{{ pretty(network.connectivity?.gemini || {}) }}</pre>
              </div>
              <div class="panel span-6">
                <h2 class="panel-title">Conectivitate Google</h2>
                <pre class="code-box compact-code">{{ pretty(network.connectivity?.google || {}) }}</pre>
              </div>
            </section>

            <section v-show="active === 'test'" class="content-grid">
              <div class="panel span-6">
                <h2 class="panel-title">Cerere</h2>
                <NForm label-placement="top"><div class="form-grid">
                  <NFormItem label="Interfață"><NSelect v-model:value="test.endpoint" :options="endpointOptions" /></NFormItem>
                  <NFormItem label="Model"><NSelect v-model:value="test.model" :options="modelOptions" filterable tag /></NFormItem>
                  <NFormItem label="Ieșire streaming"><NSwitch v-model:value="test.stream" :disabled="test.endpoint !== 'chat'" /></NFormItem>
                  <NFormItem label="Metodă de apel"><NInput :value="currentEndpoint?.label || ''" readonly /></NFormItem>
                  <NFormItem label="Prompt" class="full"><NInput v-model:value="test.prompt" type="textarea" :autosize="{ minRows: 8, maxRows: 16 }" /></NFormItem>
                </div></NForm>
                <div class="button-row"><NButton type="primary" :loading="testing" @click="runTest"><template #icon><NIcon :component="PlayOutline" /></template>Rulează test</NButton><NButton secondary @click="copyText(curlCommand, 'curl copiat')"><template #icon><NIcon :component="ClipboardOutline" /></template>Copiază curl</NButton><NButton secondary @click="test.result = ''"><template #icon><NIcon :component="TrashOutline" /></template>Golire</NButton></div>
              </div>
              <div class="panel span-6"><h2 class="panel-title">Răspuns</h2><pre class="code-box result-box">{{ test.result || 'Așteptare rezultat test' }}</pre></div>
              <div class="panel span-12"><div class="panel-head"><h2 class="panel-title">Metodă de apel</h2><NButton text type="primary" @click="copyText(callGuide, 'Instrucțiuni de apel copiate')"><template #icon><NIcon :component="CopyOutline" /></template>Copiază instrucțiuni</NButton></div><pre class="code-box compact-code">{{ callGuide }}</pre></div>
            </section>

            <section v-show="active === 'settings'" class="content-grid">
              <div class="panel span-12 settings-panel">
                <h2 class="panel-title">Configurări</h2>
                <NForm label-placement="top">
                  <div class="form-grid settings-grid">
                    <NFormItem label="Chei API" class="full">
                      <div class="secret-manager">
                        <div class="inline-editor"><NInput v-model:value="apiKeyDraft" type="password" show-password-on="click" placeholder="Introduceți cheia API" @keyup.enter="addApiKey" /><NButton type="primary" @click="addApiKey">{{ editingApiKeyIndex === null ? 'Adaugă' : 'Salvează' }}</NButton></div>
                        <table class="secret-table"><thead><tr><th>Nr. crt.</th><th>Cheie</th><th>Acțiuni</th></tr></thead><tbody><tr v-if="!apiKeyItems.length"><td colspan="3" class="empty-cell">Chei dezactivate</td></tr><tr v-for="(item, index) in apiKeyItems" :key="`${item}-${index}`" @click="copyApiKey(index)"><td>{{ index + 1 }}</td><td class="secret-value">{{ editingApiKeyIndex === index ? item : maskSecret(item) }}</td><td><NSpace size="small" @click.stop><NButton size="small" secondary @click="editApiKey(index)">Modifică</NButton><NButton size="small" tertiary type="error" @click="deleteApiKey(index)">Șterge</NButton></NSpace></td></tr></tbody></table>
                      </div>
                    </NFormItem>
                    <NFormItem label="Cookie" class="full">
                      <div class="secret-manager">
                        <NInput v-model:value="cookieDraft" type="textarea" placeholder="Lipiți Cookie-ul complet; puteți adăuga mai multe, primul este valid implicit" :autosize="{ minRows: 2, maxRows: 5 }" />
                        <div class="button-row tight"><NButton type="primary" @click="addCookie">{{ editingCookieIndex === null ? 'Adaugă Cookie' : 'Salvează Cookie' }}</NButton><NButton secondary @click="cookieDraft = ''; editingCookieIndex = null">Anulează</NButton></div>
                        <table class="secret-table"><thead><tr><th>Nr. crt.</th><th>Stare</th><th>Acțiuni</th></tr></thead><tbody><tr v-if="!cookieItems.length"><td colspan="3" class="empty-cell">Cookie neconfigurat</td></tr><tr v-for="(item, index) in cookieItems" :key="`${item.path}-${index}`"><td>{{ index + 1 }}</td><td class="secret-value">{{ item.content ? maskSecret(item.content) : (item.path || 'De salvat') }}</td><td><NSpace size="small"><NButton size="small" secondary @click="editCookie(index)">Modifică</NButton><NButton size="small" tertiary type="error" @click="deleteCookie(index)">Șterge</NButton></NSpace></td></tr></tbody></table>
                      </div>
                    </NFormItem>
                    <NFormItem label="Parolă nouă administrator"><NInput v-model:value="config.admin_password" type="password" show-password-on="click" placeholder="Lăsați gol pentru a nu modifica; implicit sk-admin" /></NFormItem>
                    <NFormItem label="Proxy"><NInput v-model:value="config.proxy" placeholder="De ex. http://127.0.0.1:7890" /></NFormItem>
                    <NFormItem label="Model implicit"><NSelect v-model:value="config.default_model" :options="modelOptions" filterable tag /></NFormItem>
                    <NFormItem label="Forțează ieșire non-streaming"><NSwitch v-model:value="config.force_non_stream" /></NFormItem>
                    <NFormItem label="URL de bază public"><NInput v-model:value="config.public_base_url" placeholder="De ex. https://your-project.vercel.app/v1" /></NFormItem>
                    <NFormItem label="Text de rezervă pentru răspuns gol"><NInput v-model:value="config.empty_response_fallback" type="textarea" :autosize="{ minRows: 2, maxRows: 4 }" /></NFormItem>
                  </div>
                </NForm>
                <div class="button-row"><NButton type="primary" :loading="saving" @click="saveConfig"><template #icon><NIcon :component="SaveOutline" /></template>Salvează configurările</NButton><NButton secondary @click="loadStatus(true)"><template #icon><NIcon :component="RefreshOutline" /></template>Recitire</NButton></div>
              </div>
              <div class="panel span-12"><h2 class="panel-title">Configurări curente</h2><pre class="code-box compact-code">{{ pretty({ ...config, cookie_content: '', cookie_contents: cookieItems.map((item) => item.content ? 'De actualizat' : item.path).filter(Boolean), admin_password: config.admin_password ? 'De actualizat' : '', api_keys: apiKeyItems }) }}</pre></div>
            </section>

            <section v-show="active === 'logs'" class="content-grid">
              <div class="panel span-12">
                <div class="panel-head"><h2 class="panel-title">Jurnale de rulare</h2><NSpace align="center" wrap><NCheckbox v-model:checked="stickToBottom">Derulare automată</NCheckbox><NSwitch v-model:value="autoLogs" /><NButton secondary @click="readLogs(true)"><template #icon><NIcon :component="RefreshOutline" /></template>Reîncărcare</NButton><NButton secondary @click="copyText(logs.text, 'Jurnal copiat')"><template #icon><NIcon :component="DocumentTextOutline" /></template>Copiază</NButton><NButton secondary @click="logs.text = ''"><template #icon><NIcon :component="TrashOutline" /></template>Golește vizualizarea</NButton></NSpace></div>
                <pre v-if="logs.text" ref="logBox" class="code-box log-box">{{ logs.text }}</pre><NEmpty v-else description="Niciun jurnal momentan" />
              </div>
            </section>
          </NSpin>
        </main>
      </div>
    </NMessageProvider>
  </NConfigProvider>
</template>