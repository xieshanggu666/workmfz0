<script setup>
import { ref, computed, reactive } from 'vue'
import { useEventStore } from '@/store/event'
const store = useEventStore()

const toast = ref('')
const showToast = (msg, kind = 'ok') => { toast.value = { msg, kind }; setTimeout(() => toast.value = '', 3600) }
const tab = ref('gate')
const tabs = [
  ['gate', '🛂', '闸机核验'], ['creds', '🪪', '证件台账'], ['logs', '📒', '核验流水'],
  ['exceptions', '🚨', '异常审计'], ['staff', '🧰', '工作人员']
]

const ov = computed(() => store.accessOverview || { by_type: {}, today: {}, checkin: {} })
const HOLDER_LABEL = { team: '参赛队', athlete: '运动员', referee: '裁判员', staff: '工作人员' }
const HOLDER_ICON = { team: '🏀', athlete: '🏃', referee: '🧑‍⚖️', staff: '🧰' }
const CRED_STATUS = {
  active: { tag: 'g', text: '有效' }, suspended: { tag: 'y', text: '停用' }, revoked: { tag: 'r', text: '注销' }
}
const REASON_META = {
  OK: { tag: 'g', text: '放行' }, ALREADY_IN: { tag: 'y', text: '重复入场' }, SUSPENDED: { tag: 'r', text: '证件停用' },
  REVOKED: { tag: 'r', text: '证件注销' }, EXPIRED: { tag: 'r', text: '已过期' }, NOT_YET: { tag: 'y', text: '未生效' },
  ZONE_DENIED: { tag: 'r', text: '区域未授权' }, VENUE_DENIED: { tag: 'r', text: '场地不符' },
  OUTSIDE_WINDOW: { tag: 'y', text: '非入场时间窗' }, NO_SCHEDULE: { tag: 'y', text: '无赛程/排班' }, UNKNOWN: { tag: 'r', text: '查无此证' }
}

/* ---------------- 闸机核验 ---------------- */
const nowHM = () => { const d = new Date(); return `${String(d.getHours()).padStart(2, '0')}:${String(d.getMinutes()).padStart(2, '0')}` }
const form = reactive({ code: '', venue_id: null, zone: '', direction: 'in', now: nowHM(), gate: '' })
const resultBox = ref(null)
const zonesByType = t => ({
  team: ['运动员区', '热身区'], athlete: ['运动员区', '检录区', '热身区'],
  referee: ['裁判区', '比赛区', '记录台'], staff: ['工作区', '后勤区']
}[t] || [])
async function check() {
  if (!form.venue_id) return showToast('请选择核验点（场地）', 'err')
  try {
    const r = await store.checkAccess({
      code: form.code, venue_id: Number(form.venue_id), zone: form.zone || null,
      direction: form.direction, now: form.now || null, gate: form.gate || null
    })
    resultBox.value = r
    if (r.ok) showToast(r.reason === 'ALREADY_IN' ? '⚠️ 重复入场（已在场内）' : '✅ 核验通过', r.reason === 'ALREADY_IN' ? 'warn' : 'ok')
    else showToast('⛔ ' + (r.detail || r.reason_text), 'err')
    form.code = ''
  } catch (e) { showToast('⚠️ ' + e.message, 'err') }
}
// 快捷填充演示证件号
function fillCode(code) { form.code = code; if (!form.venue_id) form.venue_id = store.venues[0]?.id ?? null }
const demoCreds = computed(() => store.credentials.filter(c => c.status === 'active').slice(0, 8))

/* ---------------- 证件台账 ---------------- */
const credTypeFilter = ref('all')
const credStatusFilter = ref('all')
const credList = computed(() => store.credentials.filter(c => {
  if (credTypeFilter.value !== 'all' && c.holder_type !== credTypeFilter.value) return false
  if (credStatusFilter.value !== 'all' && c.status !== credStatusFilter.value) return false
  return true
}))
const actingCred = ref(null)
const credNote = ref('')
async function suspend(c) {
  try { await store.suspendCredential(c.id, credNote.value); showToast('已停用证件「' + c.code + '」，闸机将拦截', 'warn'); actingCred.value = null; credNote.value = '' }
  catch (e) { showToast(e.message, 'err') }
}
async function restore(c) {
  try { await store.restoreCredential(c.id); showToast('证件「' + c.code + '」已恢复有效') }
  catch (e) { showToast(e.message, 'err') }
}
async function syncAll() {
  try { const r = await store.syncCredentials('all'); showToast(`补签完成，新签发 ${r.issued} 张证件`) }
  catch (e) { showToast(e.message, 'err') }
}
function holderDesc(c) {
  const h = c.holder || {}
  if (c.holder_type === 'team' || c.holder_type === 'athlete') return `${h.unit || ''} · ${h.sport || ''}`
  if (c.holder_type === 'referee') return `${h.sport || '综合执法'} · ${h.level || ''}`
  return `${h.venue_name || '未分配场地'} · ${h.role || '场地保障'}`
}

/* ---------------- 核验流水 / 异常审计 ---------------- */
const logsList = computed(() => store.accessLogs)
const denyList = computed(() => store.accessLogs.filter(l => l.result === 'DENY'))
const pendingDeny = computed(() => denyList.value.filter(l => !l.handled))
const handlingId = ref(null)
const handleNote = ref('')
async function handle(l) {
  try { await store.handleAccessLog(l.id, handleNote.value, '安保组'); showToast('异常已处置并归档'); handlingId.value = null; handleNote.value = '' }
  catch (e) { showToast(e.message, 'err') }
}
function writebackText(l) {
  if (!l.writeback || !l.writeback.length) return '—'
  return l.writeback.map(w => w.label).join('；')
}

/* ---------------- 工作人员 ---------------- */
const stForm = reactive({ name: '', role: '安保', venue_id: null, phone: '' })
async function addStaff() {
  if (!stForm.name.trim()) return showToast('请输入姓名', 'err')
  try { await store.addStaff({ name: stForm.name.trim(), role: stForm.role, venue_id: stForm.venue_id, phone: stForm.phone }); showToast('工作人员已录入并签发证件'); stForm.name = ''; stForm.phone = '' }
  catch (e) { showToast(e.message, 'err') }
}
async function toggleStaff(st) {
  const status = st.status === '在岗' ? '离岗' : '在岗'
  try { await store.updateStaff(st.id, { status }); showToast(status === '离岗' ? '已离岗，证件同步停用' : '已回岗，证件恢复') }
  catch (e) { showToast(e.message, 'err') }
}
</script>

<template>
  <div v-if="store.loaded">
    <div class="page-h">
      <div><h2>🪪 赛事证件与入场核验</h2><div class="sub">运动员 · 裁判员 · 场地工作人员按资格与赛程获得动态权限 · 核验结果回写报名/排班/比赛/检录/异常审计</div></div>
      <div v-if="toast" class="toast" :class="{ deny: toast.kind === 'err', warn: toast.kind === 'warn' }">{{ toast.msg }}</div>
      <div class="row">
        <button class="btn ghost sm" @click="syncAll">🔄 补签缺证</button>
        <button class="btn ghost sm" @click="store.refresh">刷新</button>
      </div>
    </div>

    <!-- 统计卡 -->
    <div class="grid" style="grid-template-columns:repeat(6,1fr)">
      <div class="card stat"><span class="bar" style="background:linear-gradient(90deg,#2f9bff,#7cc4ff)"></span><span class="ic">🪪</span><b>{{ ov.issued }}</b><em>证件总数（有效 {{ ov.active }}）</em></div>
      <div class="card stat" v-for="t in ['team','athlete','referee','staff']" :key="t">
        <span class="bar" style="background:linear-gradient(90deg,#22c15e,#7edda4)"></span>
        <span class="ic">{{ HOLDER_ICON[t] }}</span>
        <b>{{ ov.by_type?.[t]?.active || 0 }}<span style="font-size:13px;color:var(--muted)"> / {{ ov.by_type?.[t]?.issued || 0 }}</span></b>
        <em>{{ HOLDER_LABEL[t] }}有效/签发</em>
      </div>
      <div class="card stat"><span class="bar" style="background:linear-gradient(90deg,#e5484d,#f0a1a1)"></span><span class="ic">🚨</span><b>{{ pendingDeny.length }}</b><em>待处置异常拦截</em></div>
    </div>

    <!-- 回写概览 -->
    <div class="card mt">
      <div class="caption">🔁 核验结果回写动态 <span class="hint">放行核验实时回写各业务环节</span></div>
      <div class="pad row" style="gap:26px;flex-wrap:wrap">
        <span class="wb-chip">🏀 球队到场场次 <b>{{ ov.checkin?.teams || 0 }}</b></span>
        <span class="wb-chip">🧑‍⚖️ 裁判签到排班 <b>{{ ov.checkin?.referees || 0 }}</b></span>
        <span class="wb-chip">🏃 田径检录人次 <b>{{ ov.checkin?.athletes || 0 }}</b></span>
        <span class="wb-chip">🧰 工作人员到岗 <b>{{ ov.checkin?.staff || 0 }}</b></span>
        <span class="wb-chip">✅ 今日放行 <b>{{ ov.today?.pass || 0 }}</b></span>
        <span class="wb-chip" :class="{ alert: (ov.today?.deny || 0) > 0 }">⛔ 今日拦截 <b>{{ ov.today?.deny || 0 }}</b></span>
      </div>
    </div>

    <!-- 子页签 -->
    <div class="filters mt16">
      <button v-for="t in tabs" :key="t[0]" class="chip" :class="{ on: tab === t[0] }" @click="tab = t[0]">
        {{ t[1] }} {{ t[2] }}
        <span v-if="t[0] === 'exceptions' && pendingDeny.length" class="chip-badge">{{ pendingDeny.length }}</span>
      </button>
    </div>

    <!-- ============ 闸机核验 ============ -->
    <template v-if="tab === 'gate'">
      <div class="grid g2 mt16">
        <div class="card">
          <div class="caption">🛂 闸机 / 核验点刷卡</div>
          <div class="pad">
            <div class="form-row">
              <label>证件号</label>
              <input v-model="form.code" placeholder="如 TM-0003 / RF-0002 / ST-0001" @keyup.enter="check" style="font-family:Consolas,monospace;font-weight:700;letter-spacing:1px">
            </div>
            <div class="form-row">
              <label>核验点</label>
              <select v-model.number="form.venue_id">
                <option :value="null" disabled>选择场地</option>
                <option v-for="v in store.venues" :key="v.id" :value="v.id">{{ v.name }}</option>
              </select>
            </div>
            <div class="form-row">
              <label>功能区域</label>
              <select v-model="form.zone">
                <option value="">不限区域（场地核验）</option>
                <option v-for="z in ['运动员区','热身区','检录区','裁判区','比赛区','记录台','工作区','后勤区','贵宾区']" :key="z" :value="z">{{ z }}</option>
              </select>
            </div>
            <div class="form-row">
              <label>方向</label>
              <select v-model="form.direction">
                <option value="in">⬇ 入场</option>
                <option value="out">⬆ 离场</option>
              </select>
              <label style="width:auto;margin-left:10px">核验时刻</label>
              <input v-model="form.now" type="time" style="max-width:120px">
              <span class="hint" style="font-size:11px">默认当前；赛程窗口＝开赛前 90 分~开赛后 60 分</span>
            </div>
            <div class="form-row">
              <label>核验点/人</label>
              <input v-model="form.gate" placeholder="如：篮球馆北闸 · 安保老张（选填）">
            </div>
            <div class="row mt8" style="justify-content:flex-end">
              <button class="btn primary" @click="check">🎫 核验</button>
            </div>
            <div class="hint mt8" style="font-size:11px;line-height:1.8">
              快捷填充演示证件：
              <span v-for="c in demoCreds" :key="c.id" class="quick-code" @click="fillCode(c.code)">{{ c.code }}</span>
            </div>
          </div>
        </div>

        <!-- 核验结论 -->
        <div class="card">
          <div class="caption">📟 核验结论与回写</div>
          <div class="pad">
            <div v-if="!resultBox" class="empty">请在左侧刷卡核验，结论将在此显示</div>
            <template v-else>
              <div class="result-box" :class="resultBox.ok ? (resultBox.reason === 'ALREADY_IN' ? 'warn' : 'pass') : 'deny'">
                <div class="rb-icon">{{ resultBox.ok ? (resultBox.reason === 'ALREADY_IN' ? '⚠️' : (resultBox.direction === 'out' ? '⬆️' : '✅')) : '⛔' }}</div>
                <div>
                  <div class="rb-title">{{ resultBox.ok ? (resultBox.reason === 'ALREADY_IN' ? '重复入场' : resultBox.direction === 'out' ? '离场放行' : '放行') : '禁止入场' }}</div>
                  <div class="rb-detail">{{ resultBox.detail }}</div>
                </div>
              </div>
              <table v-if="resultBox.credential" class="mt8">
                <tbody>
                  <tr><td class="k">持证人</td><td><b>{{ HOLDER_ICON[resultBox.credential.holder_type] }} {{ resultBox.credential.name }}</b>（{{ HOLDER_LABEL[resultBox.credential.holder_type] }}）</td></tr>
                  <tr><td class="k">核验点</td><td>{{ resultBox.venue?.name }}</td></tr>
                  <tr><td class="k">命中赛程/岗位</td><td>{{ resultBox.scope?.title || '—' }}<span v-if="resultBox.scope?.time_label" class="tag b" style="margin-left:6px">{{ resultBox.scope.time_label }}</span></td></tr>
                  <tr><td class="k">授权区域</td><td><span v-for="z in resultBox.credential.zones" :key="z" class="tag gray" style="margin:2px">{{ z }}</span></td></tr>
                </tbody>
              </table>
              <div v-if="resultBox.writeback?.length" class="writeback-box mt8">
                <div class="wb-cap">🔁 已实时回写</div>
                <div v-for="(w, i) in resultBox.writeback" :key="i" class="wb-line">✏️ {{ w.label }}</div>
              </div>
            </template>
          </div>
        </div>
      </div>

      <!-- 最近核验 -->
      <div class="card mt16">
        <div class="caption">🕒 最近核验记录</div>
        <div class="pad">
          <table>
            <thead><tr><th>时间</th><th>证件号</th><th>持证人</th><th>核验点</th><th>方向</th><th>结论</th><th>命中赛程/范围</th></tr></thead>
            <tbody>
              <tr v-for="l in logsList.slice(0, 8)" :key="l.id">
                <td class="mono">{{ l.created_at }}</td>
                <td class="mono" style="font-weight:700">{{ l.code }}</td>
                <td>{{ l.holder_name || '—' }}</td>
                <td>{{ l.venue_name }}{{ l.zone ? ' · ' + l.zone : '' }}</td>
                <td>{{ l.direction === 'out' ? '离场' : '入场' }}</td>
                <td><span class="tag" :class="REASON_META[l.reason]?.tag || 'gray'">{{ REASON_META[l.reason]?.text || l.reason }}</span></td>
                <td class="ph" style="max-width:280px;overflow:hidden;text-overflow:ellipsis;white-space:nowrap">{{ l.detail }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </template>

    <!-- ============ 证件台账 ============ -->
    <template v-else-if="tab === 'creds'">
      <div class="filters mt16">
        <button class="chip" :class="{ on: credTypeFilter === 'all' }" @click="credTypeFilter = 'all'">全部类型</button>
        <button v-for="(lab, t) in HOLDER_LABEL" :key="t" class="chip" :class="{ on: credTypeFilter === t }" @click="credTypeFilter = t">{{ HOLDER_ICON[t] }} {{ lab }}</button>
        <span style="width:14px"></span>
        <button class="chip" :class="{ on: credStatusFilter === 'all' }" @click="credStatusFilter = 'all'">全部状态</button>
        <button v-for="(meta, key) in CRED_STATUS" :key="key" class="chip" :class="{ on: credStatusFilter === key }" @click="credStatusFilter = key">{{ meta.text }}</button>
      </div>
      <div class="card mt16">
        <div class="caption">🪪 证件台账 <span class="hint">共 {{ credList.length }} 张 · 权限随资格/赛程/排班/岗位动态变化</span></div>
        <div class="pad" style="display:flex;flex-direction:column;gap:10px">
          <div v-for="c in credList" :key="c.id" class="cred-card">
            <div class="cred-top">
              <div class="row" style="gap:10px">
                <span class="cred-code">{{ HOLDER_ICON[c.holder_type] }} {{ c.code }}</span>
                <span class="tag" :class="CRED_STATUS[c.status].tag">{{ CRED_STATUS[c.status].text }}</span>
                <span v-if="c.status === 'active' && c.open_now" class="tag g">🟢 当前可入场</span>
                <span v-else-if="c.status === 'active'" class="tag y">⏸ 暂无开放赛程/岗位</span>
              </div>
              <div class="row" style="gap:6px">
                <template v-if="c.status === 'active'">
                  <button class="btn ghost sm" @click="actingCred = actingCred === c.id ? null : c.id; credNote = ''">⏸ 停用</button>
                </template>
                <button v-else-if="c.status === 'suspended'" class="btn green sm" @click="restore(c)">▶ 恢复</button>
                <span v-else class="hint">注销不可恢复</span>
              </div>
            </div>
            <div class="mrow">
              <span class="t"><b>{{ c.holder?.name }}</b> <span class="tag gray">{{ HOLDER_LABEL[c.holder_type] }}</span> <span class="hint">{{ holderDesc(c) }}</span></span>
              <span class="hint">签发 {{ c.issued_at }} · {{ c.issued_by }}</span>
            </div>
            <div v-if="c.suspend_reason" class="note-line">⚠️ {{ c.suspend_reason }}</div>
            <!-- 动态权限范围 -->
            <div class="scope-box">
              <div class="scope-cap">🎯 动态权限范围（{{ c.scopes.length }}）<span class="hint">实时按赛程/排班/岗位计算</span></div>
              <div v-if="!c.scopes.length" class="hint" style="padding:4px 0">当前无有效范围（资格失效或暂无待赛/排班/当班）</div>
              <div v-for="s in c.scopes" :key="s.key" class="scope-item">
                <span class="tag" :class="s.open_now ? 'g' : 'gray'">{{ s.open_now ? '🟢 窗口开放中' : (s.time_label ? '🕒 ' + s.time_label : '全天') }}</span>
                <b style="font-size:12.5px">{{ s.title }}</b>
                <span class="hint">@ {{ s.venue || '未指定场地' }}<template v-if="s.role"> · {{ { chief: '主裁', assistant: '助理裁判', recorder: '记录台' }[s.role] }}</template></span>
              </div>
            </div>
            <div class="row mt8" style="gap:5px;flex-wrap:wrap">
              <span v-for="z in c.zones" :key="z" class="tag b">🔑 {{ z }}</span>
            </div>
            <div v-if="actingCred === c.id && c.status === 'active'" class="act-panel">
              <input v-model="credNote" placeholder="停用原因（如：芯片损坏、违规暂扣）" style="flex:1;min-width:220px">
              <button class="btn ghost sm" @click="actingCred = null">取消</button>
              <button class="btn sm" style="background:#fff6dd;color:#c98a00" @click="suspend(c)">确认停用</button>
            </div>
          </div>
          <div v-if="!credList.length" class="empty">暂无证件</div>
        </div>
      </div>
    </template>

    <!-- ============ 核验流水 ============ -->
    <template v-else-if="tab === 'logs'">
      <div class="card mt16">
        <div class="caption">📒 核验流水 <span class="hint">全部放行与拦截均留痕，最近 {{ logsList.length }} 条</span></div>
        <div class="pad">
          <table>
            <thead><tr><th>时间</th><th>证件号</th><th>类型</th><th>持证人</th><th>核验点/区域</th><th>方向</th><th>结论</th><th>回写</th><th>处置</th></tr></thead>
            <tbody>
              <tr v-for="l in logsList" :key="l.id">
                <td class="mono" style="white-space:nowrap">{{ l.created_at }}</td>
                <td class="mono" style="font-weight:700">{{ l.code }}</td>
                <td>{{ HOLDER_ICON[l.holder_type] || '❓' }} {{ HOLDER_LABEL[l.holder_type] || '未知' }}</td>
                <td>{{ l.holder_name || '—' }}</td>
                <td>{{ l.venue_name || '—' }}<div class="hint">{{ l.zone || '' }} {{ l.gate ? '· ' + l.gate : '' }}</div></td>
                <td>{{ l.direction === 'out' ? '离场' : '入场' }}</td>
                <td><span class="tag" :class="REASON_META[l.reason]?.tag || 'gray'">{{ REASON_META[l.reason]?.text || l.reason }}</span><div class="hint" style="max-width:230px">{{ l.detail }}</div></td>
                <td class="ph" style="max-width:210px;font-size:12px">{{ writebackText(l) }}</td>
                <td>
                  <template v-if="l.result === 'DENY'">
                    <span v-if="l.handled" class="tag g">已处置<div class="hint" style="font-size:10px">{{ l.handled_by }} {{ l.handled_at }}</div></span>
                    <button v-else class="btn sm" style="background:#ffecec;color:#e5484d;padding:4px 9px" @click="handlingId = handlingId === l.id ? null : l.id; handleNote = ''">处置</button>
                  </template>
                  <span v-else class="tag gray">—</span>
                </td>
              </tr>
            </tbody>
          </table>
          <div v-if="!logsList.length" class="empty">暂无核验记录</div>
        </div>
      </div>
    </template>

    <!-- ============ 异常审计 ============ -->
    <template v-else-if="tab === 'exceptions'">
      <div class="grid g2 mt16">
        <div class="card" :class="{ alert_card: pendingDeny.length }">
          <div class="caption">🚨 待处置异常 <span class="hint">{{ pendingDeny.length }} 起拦截待核实</span></div>
          <div class="pad" style="display:flex;flex-direction:column;gap:10px">
            <div v-for="l in pendingDeny" :key="l.id" class="exc-card">
              <div class="mheader">
                <span class="mono" style="font-weight:800;color:#e5484d">{{ l.code || '（空号）' }}</span>
                <span class="tag r">{{ REASON_META[l.reason]?.text || l.reason }}</span>
              </div>
              <div class="mrow"><span class="t">{{ l.holder_name || '身份未知' }} @ {{ l.venue_name }}{{ l.zone ? ' · ' + l.zone : '' }}</span><span class="hint">{{ l.created_at }}</span></div>
              <div class="note-line">📝 {{ l.detail }}</div>
              <div v-if="handlingId === l.id" class="act-panel">
                <input v-model="handleNote" placeholder="处置说明（如：人证不符暂扣、走错通道引导放行）" style="flex:1;min-width:200px">
                <button class="btn ghost sm" @click="handlingId = null">取消</button>
                <button class="btn primary sm" @click="handle(l)">确认处置</button>
              </div>
              <button v-else class="btn ghost sm mt8" @click="handlingId = l.id">✔ 标记处置</button>
            </div>
            <div v-if="!pendingDeny.length" class="empty">🎉 无待处置异常，全部拦截已闭环</div>
          </div>
        </div>
        <div class="card">
          <div class="caption">📁 已处置异常归档</div>
          <div class="pad" style="display:flex;flex-direction:column;gap:10px">
            <div v-for="l in denyList.filter(x => x.handled)" :key="l.id" class="exc-card done-exc">
              <div class="mheader"><span class="mono">{{ l.code }}</span><span class="tag g">已处置 · {{ l.reason && REASON_META[l.reason]?.text }}</span></div>
              <div class="ph" style="font-size:12px">{{ l.detail }}</div>
              <div class="hint mt8">👮 {{ l.handled_by }} · {{ l.handled_at }} · {{ l.handle_note }}</div>
            </div>
            <div v-if="!denyList.filter(x => x.handled).length" class="empty">暂无已处置记录</div>
          </div>
        </div>
      </div>
    </template>

    <!-- ============ 工作人员 ============ -->
    <template v-else-if="tab === 'staff'">
      <div class="grid g2 mt16">
        <div class="card">
          <div class="caption">➕ 录入场地工作人员 <span class="hint">按所属场地获得当班通行权限</span></div>
          <div class="pad">
            <div class="form-row"><label>姓名</label><input v-model="stForm.name" placeholder="工作人员姓名"></div>
            <div class="form-row"><label>岗位</label>
              <select v-model="stForm.role">
                <option>安保</option><option>医护</option><option>器材管理</option><option>场地维护</option><option>记录播报</option><option>志愿者</option>
              </select>
            </div>
            <div class="form-row"><label>所属场地</label>
              <select v-model.number="stForm.venue_id">
                <option :value="null">不分配（无场地通行范围）</option>
                <option v-for="v in store.venues" :key="v.id" :value="v.id">{{ v.name }}</option>
              </select>
            </div>
            <div class="form-row"><label>联系电话</label><input v-model="stForm.phone" placeholder="选填"></div>
            <div class="row" style="justify-content:flex-end"><button class="btn primary" @click="addStaff">录入并签发证件</button></div>
          </div>
        </div>
        <div class="card">
          <div class="caption">🧰 工作人员名册 <span class="hint">{{ store.staff.length }} 人</span></div>
          <div class="pad">
            <table>
              <thead><tr><th>姓名</th><th>岗位</th><th>所属场地</th><th>状态</th><th>最近到岗</th><th></th></tr></thead>
              <tbody>
                <tr v-for="st in store.staff" :key="st.id">
                  <td><b>{{ st.name }}</b></td>
                  <td>{{ st.role }}</td>
                  <td>{{ st.venue_name || '—' }}</td>
                  <td><span class="tag" :class="st.status === '在岗' ? 'g' : 'gray'">{{ st.status }}</span></td>
                  <td class="hint">{{ st.last_check_at || '—' }}</td>
                  <td><button class="btn ghost sm" @click="toggleStaff(st)">{{ st.status === '在岗' ? '离岗' : '回岗' }}</button></td>
                </tr>
                <tr v-if="!store.staff.length"><td colspan="6" class="empty">暂无工作人员</td></tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </template>
  </div>
</template>

<style scoped>
.wb-chip { background: var(--bg2); border-radius: 12px; padding: 8px 14px; font-size: 13px; color: var(--muted); font-weight: 600; }
.wb-chip b { color: var(--ink); font-size: 16px; margin: 0 3px; }
.wb-chip.alert b { color: #e5484d; }
.chip-badge { background: #e5484d; color: #fff; border-radius: 9px; font-size: 10px; padding: 0 5px; margin-left: 5px; }
.form-row { display: flex; align-items: center; gap: 10px; margin-bottom: 12px; }
.form-row label { width: 72px; flex-shrink: 0; font-size: 12px; color: var(--muted); font-weight: 700; }
.form-row select, .form-row input { flex: 1; }
.quick-code { display: inline-block; margin: 2px 3px; font-family: Consolas, monospace; font-size: 11px; color: var(--accent3); background: #e8f2ff; border-radius: 6px; padding: 1px 7px; cursor: pointer; }
.quick-code:hover { background: #d3e8ff; }
.result-box { display: flex; gap: 14px; align-items: center; border-radius: 14px; padding: 18px; }
.result-box.pass { background: linear-gradient(135deg, #e9faf0, #f4fcf6); border: 1px solid #bfeccd; }
.result-box.warn { background: linear-gradient(135deg, #fff8e8, #fffdf6); border: 1px solid #f0d79a; }
.result-box.deny { background: linear-gradient(135deg, #ffefef, #fff7f7); border: 1px solid #f2bcbc; }
.rb-icon { font-size: 40px; }
.rb-title { font-size: 20px; font-weight: 800; }
.result-box.pass .rb-title { color: var(--accent2); }
.result-box.warn .rb-title { color: #c98a00; }
.result-box.deny .rb-title { color: #e5484d; }
.rb-detail { color: var(--muted); font-size: 13px; margin-top: 3px; }
td.k { color: var(--muted); font-weight: 700; width: 96px; white-space: nowrap; }
.writeback-box { background: #f3f8ff; border: 1px dashed var(--accent3); border-radius: 12px; padding: 10px 13px; }
.wb-cap { font-size: 12px; font-weight: 800; color: var(--accent3); margin-bottom: 5px; }
.wb-line { font-size: 12.5px; padding: 2px 0; }
.cred-card { border: 1px solid var(--line); border-radius: 14px; padding: 13px 15px; background: #fff; }
.cred-top { display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 8px; }
.cred-code { font-family: Consolas, monospace; font-weight: 800; font-size: 15px; color: var(--accent); }
.scope-box { margin-top: 9px; background: var(--bg2); border-radius: 10px; padding: 9px 12px; }
.scope-cap { font-size: 12px; font-weight: 800; margin-bottom: 5px; }
.scope-item { display: flex; align-items: center; gap: 8px; padding: 3px 0; flex-wrap: wrap; }
.note-line { font-size: 12px; color: var(--muted); margin-top: 6px; padding: 5px 9px; background: var(--bg2); border-radius: 8px; }
.act-panel { display: flex; gap: 8px; margin-top: 10px; padding: 10px; background: var(--bg2); border-radius: 10px; align-items: center; flex-wrap: wrap; }
.exc-card { border: 1px solid #f2d3d3; border-radius: 12px; padding: 11px 13px; background: #fffafa; }
.exc-card.done-exc { border-color: #c9eecf; background: #f7fdf9; }
.toast.deny { background: #e5484d; }
.toast.warn { background: #c98a00; }
</style>
