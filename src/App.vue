<script setup lang="ts">
import { computed, reactive, ref, watch } from "vue";
import { ElMessage, ElMessageBox } from "element-plus";

type TempZone = "冷藏" | "常温";
type OrderStatus = "候装" | "已装" | "已出发";
type RiderState = "在岗" | "请假" | "已出发";

interface CommunityOrder {
  id: string;
  batch: string; // 批次
  community: string; // 社区
  zone: TempZone; // 温区
  slot: string; // 承诺时段
  qty: number; // 件数
  status: OrderStatus;
  riderId: string | null;
  holdReason: string; // 候装区滞留原因
  createdAt: string;
}

interface Rider {
  id: string;
  name: string;
  shift: string; // 班次
  bagClean: boolean; // 保温袋是否未污染
  capacity: number; // 容量（件）
  berth: string; // 泊位，请假交接后当天释放
  state: RiderState;
}

interface HandoverLog {
  id: string;
  time: string;
  fromRider: string;
  toRider: string;
  movedQty: number; // 已装数量交接
  returnedQty: number; // 装不下退回候装区数量
  detail: string;
}

interface DockState {
  date: string;
  orders: CommunityOrder[];
  riders: Rider[];
  handovers: HandoverLog[];
}

const STORAGE_KEY = "hxwlfront-15-loading-dock-v1";
const SHIFTS = ["晚班A", "晚班B"];
const SLOTS = ["17:00-19:00", "19:00-21:00", "次日08:00-10:00"];
const ZONES: TempZone[] = ["冷藏", "常温"];

function pad(value: number) {
  return String(value).padStart(2, "0");
}

function todayStr() {
  const d = new Date();
  return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}`;
}

function timeNow() {
  const d = new Date();
  return `${pad(d.getHours())}:${pad(d.getMinutes())}`;
}

function seedState(): DockState {
  // 演示数据：覆盖冷藏/常温混装拦截、容量不足滞留、出发与请假交接场景
  const riders: Rider[] = [
    { id: "seed-r1", name: "赵大勇", shift: "晚班A", bagClean: true, capacity: 12, berth: "B-01", state: "在岗" },
    { id: "seed-r2", name: "钱小妹", shift: "晚班A", bagClean: false, capacity: 10, berth: "B-02", state: "在岗" },
    { id: "seed-r3", name: "孙铁柱", shift: "晚班B", bagClean: false, capacity: 14, berth: "B-03", state: "在岗" },
    { id: "seed-r4", name: "周凯", shift: "晚班B", bagClean: true, capacity: 8, berth: "B-04", state: "已出发" }
  ];
  const now = new Date().toISOString();
  const orders: CommunityOrder[] = [
    { id: "seed-o1", batch: "T0926-晚团1", community: "阳光花园", zone: "冷藏", slot: "19:00-21:00", qty: 4, status: "候装", riderId: null, holdReason: "", createdAt: now },
    { id: "seed-o2", batch: "T0926-晚团1", community: "阳光花园", zone: "常温", slot: "19:00-21:00", qty: 6, status: "候装", riderId: null, holdReason: "", createdAt: now },
    { id: "seed-o3", batch: "T0926-晚团2", community: "滨江苑", zone: "冷藏", slot: "17:00-19:00", qty: 5, status: "候装", riderId: null, holdReason: "", createdAt: now },
    { id: "seed-o4", batch: "T0926-晚团2", community: "滨江苑", zone: "常温", slot: "17:00-19:00", qty: 8, status: "候装", riderId: null, holdReason: "", createdAt: now },
    { id: "seed-o5", batch: "T0926-晚团3", community: "幸福里", zone: "冷藏", slot: "次日08:00-10:00", qty: 6, status: "候装", riderId: null, holdReason: "容量不足：同班次冷藏保温袋余量均小于6件", createdAt: now },
    { id: "seed-o6", batch: "T0926-晚团3", community: "幸福里", zone: "冷藏", slot: "次日08:00-10:00", qty: 3, status: "已装", riderId: "seed-r1", holdReason: "", createdAt: now },
    { id: "seed-o7", batch: "T0926-晚团3", community: "幸福里", zone: "常温", slot: "次日08:00-10:00", qty: 7, status: "已装", riderId: "seed-r2", holdReason: "", createdAt: now },
    { id: "seed-o8", batch: "T0926-晚团3", community: "和平社区", zone: "常温", slot: "19:00-21:00", qty: 10, status: "已装", riderId: "seed-r3", holdReason: "", createdAt: now },
    { id: "seed-o9", batch: "T0926-晚团2", community: "滨江苑", zone: "冷藏", slot: "17:00-19:00", qty: 3, status: "已出发", riderId: "seed-r4", holdReason: "", createdAt: now },
    { id: "seed-o10", batch: "T0926-晚团2", community: "滨江苑", zone: "冷藏", slot: "17:00-19:00", qty: 2, status: "已出发", riderId: "seed-r4", holdReason: "", createdAt: now }
  ];
  return { date: todayStr(), riders, orders, handovers: [] };
}

function loadState(): DockState {
  const raw = localStorage.getItem(STORAGE_KEY);
  if (raw) {
    try {
      const parsed = JSON.parse(raw) as DockState;
      if (parsed.date === todayStr() && Array.isArray(parsed.orders) && Array.isArray(parsed.riders)) {
        return parsed;
      }
    } catch {
      // 数据损坏则重新播种
    }
  }
  return seedState();
}

const state = reactive<DockState>(loadState());

watch(
  state,
  () => localStorage.setItem(STORAGE_KEY, JSON.stringify(state)),
  { deep: true }
);

const tab = ref("dock");

const orderForm = reactive<{ batch: string; community: string; zone: TempZone; slot: string; qty: number }>({
  batch: "",
  community: "",
  zone: "冷藏",
  slot: SLOTS[1],
  qty: 3
});

const riderForm = reactive<{ name: string; shift: string; capacity: number; berth: string; bagClean: boolean }>({
  name: "",
  shift: SHIFTS[0],
  capacity: 10,
  berth: "",
  bagClean: true
});

// 候装单 -> 目标骑手；请假骑手 -> 接管人
const targets = reactive<Record<string, string>>({});
const takeovers = reactive<Record<string, string>>({});

function riderById(id: string | null) {
  return state.riders.find((r) => r.id === id) ?? null;
}

function riderName(id: string | null) {
  return riderById(id)?.name ?? "候装区";
}

function riderLoadedOrders(riderId: string) {
  return state.orders.filter((o) => o.riderId === riderId && o.status === "已装");
}

function usedQty(riderId: string) {
  return riderLoadedOrders(riderId).reduce((sum, o) => sum + o.qty, 0);
}

function loadedZone(riderId: string): TempZone | null {
  return riderLoadedOrders(riderId)[0]?.zone ?? null;
}

function capacityPercent(r: Rider) {
  return Math.min(100, Math.round((usedQty(r.id) / Math.max(r.capacity, 1)) * 100));
}

const onDutyRiders = computed(() => state.riders.filter((r) => r.state === "在岗"));

const waitingOrders = computed(() =>
  [...state.orders].filter((o) => o.status === "候装").sort((a, b) => a.createdAt.localeCompare(b.createdAt))
);

const loadedOrders = computed(() => state.orders.filter((o) => o.status === "已装"));

const departedOrders = computed(() => state.orders.filter((o) => o.status === "已出发"));

const activeLoadedRiders = computed(() =>
  state.riders.filter((r) => r.state === "在岗" && riderLoadedOrders(r.id).length > 0)
);

const metrics = computed(() => {
  const sum = (list: CommunityOrder[]) => list.reduce((acc, o) => acc + o.qty, 0);
  return [
    { label: "今日到站（件）", value: sum(state.orders) },
    { label: "候装滞留（件）", value: sum(waitingOrders.value) },
    { label: "已装待发（件）", value: sum(loadedOrders.value) },
    { label: "已出发（件）", value: sum(departedOrders.value) }
  ];
});

const batchCount = computed(() => new Set(state.orders.map((o) => o.batch)).size);

/** 装车校验：返回 null 表示可装，否则返回滞留/拒收原因 */
function loadBlockReason(order: CommunityOrder, rider: Rider): string | null {
  if (rider.state === "请假") return "骑手已请假";
  if (rider.state === "已出发") return "骑手已出发离站";
  const zone = loadedZone(rider.id);
  if (zone && zone !== order.zone) {
    return `保温袋已装${zone}件，不得与${order.zone}件混装`;
  }
  if (order.zone === "冷藏" && !rider.bagClean) {
    return "保温袋已被常温件污染，冷藏件只能使用未污染保温袋";
  }
  const remain = rider.capacity - usedQty(rider.id);
  if (remain < order.qty) {
    return `容量不足：余量${remain}件，本单${order.qty}件`;
  }
  return null;
}

function applyLoad(order: CommunityOrder, rider: Rider): string | null {
  const reason = loadBlockReason(order, rider);
  if (reason) return reason;
  const contaminatedNow = order.zone === "常温" && rider.bagClean && usedQty(rider.id) === 0;
  order.riderId = rider.id;
  order.status = "已装";
  order.holdReason = "";
  if (contaminatedNow) {
    rider.bagClean = false; // 常温件入袋即污染，冷藏件此后禁装
    return `已装${order.qty}件；该袋开始装常温件，标记为已污染`;
  }
  return `已装${order.qty}件`;
}

function loadOrder(order: CommunityOrder) {
  const rider = riderById(targets[order.id] ?? "");
  if (!rider) {
    ElMessage.warning("请先选择装袋骑手");
    return;
  }
  const result = applyLoad(order, rider);
  if (order.status === "已装") {
    ElMessage.success(`${rider.name}：${result}`);
  } else {
    order.holdReason = result ?? "无法装车";
    ElMessage.error(`${rider.name}无法装载：${order.holdReason}，已留在候装区`);
  }
}

/** 智能配载：按到站顺序为候装单依次寻找可装骑手，装不下写明原因 */
function autoLoad() {
  let loadedCount = 0;
  let heldCount = 0;
  for (const order of waitingOrders.value) {
    const candidates = state.riders.filter((r) => r.state === "在岗");
    let done = false;
    const reasons: string[] = [];
    for (const rider of candidates) {
      const reason = loadBlockReason(order, rider);
      if (!reason) {
        applyLoad(order, rider);
        loadedCount += 1;
        done = true;
        break;
      }
      reasons.push(reason);
    }
    if (!done) {
      order.holdReason = `全员无法装载：${[...new Set(reasons)].join("；") || "无在岗骑手"}`;
      heldCount += 1;
    }
  }
  if (loadedCount === 0 && heldCount === 0) {
    ElMessage.info("候装区没有待装订单");
  } else {
    ElMessage.success(`智能配载完成：装车${loadedCount}单，滞留候装区${heldCount}单（已写明原因）`);
  }
}

function unloadOrder(order: CommunityOrder) {
  const name = riderName(order.riderId);
  order.status = "候装";
  order.riderId = null;
  order.holdReason = `由${name}卸回候装区`;
  ElMessage.info(`订单已从${name}袋中卸回候装区`);
}

function removeOrder(order: CommunityOrder) {
  state.orders = state.orders.filter((o) => o.id !== order.id);
  delete targets[order.id];
  ElMessage.info("订单已删除");
}

async function depart(rider: Rider) {
  const mine = riderLoadedOrders(rider.id);
  if (mine.length === 0) {
    ElMessage.warning("保温袋为空，不能出发");
    return;
  }
  try {
    await ElMessageBox.confirm(
      `确认 ${rider.name} 携带 ${mine.reduce((s, o) => s + o.qty, 0)} 件出发？出发后订单不可再转单。`,
      "确认出发",
      { confirmButtonText: "出发", cancelButtonText: "取消", type: "warning" }
    );
  } catch {
    return;
  }
  for (const o of mine) o.status = "已出发";
  rider.state = "已出发";
  ElMessage.success(`${rider.name}已出发`);
}

function shiftCandidates(rider: Rider) {
  return state.riders.filter((r) => r.shift === rider.shift && r.state === "在岗" && r.id !== rider.id);
}

async function takeLeave(rider: Rider) {
  const target = riderById(takeovers[rider.id] ?? "");
  const mine = state.orders.filter((o) => o.riderId === rider.id && o.status !== "已出发");
  if (!target) {
    ElMessage.warning("请选择同班次接管骑手");
    return;
  }
  try {
    await ElMessageBox.confirm(
      `确认 ${rider.name} 请假？未出发的 ${mine.length} 单将交由同班次 ${target.name} 接管，装不下的退回候装区，原泊位 ${rider.berth || "已释放"} 当天释放。`,
      "请假交接",
      { confirmButtonText: "确认交接", cancelButtonText: "取消", type: "warning" }
    );
  } catch {
    return;
  }

  let movedQty = 0;
  let returnedQty = 0;
  const returnReasons: string[] = [];
  for (const order of mine) {
    const reason = loadBlockReason(order, target);
    if (reason) {
      order.status = "候装";
      order.riderId = null;
      order.holdReason = `交接退回：${target.name}无法接收（${reason}）`;
      returnedQty += order.qty;
      returnReasons.push(`${order.batch}·${order.community}${order.qty}件：${reason}`);
    } else {
      applyLoad(order, target);
      movedQty += order.qty;
    }
  }

  const oldBerth = rider.berth;
  rider.state = "请假";
  rider.berth = ""; // 原泊位当天释放
  delete takeovers[rider.id];

  const detailParts = [
    `${rider.name}请假，同班次${target.name}接管未出发订单`,
    `交接已装${movedQty}件`,
    returnedQty > 0 ? `退回候装区${returnedQty}件（${returnReasons.join("；")}）` : "无退回",
    `原泊位${oldBerth || "无"}当天已释放`
  ];
  state.handovers.unshift({
    id: `ho-${Date.now().toString(36)}`,
    time: `${state.date} ${timeNow()}`,
    fromRider: rider.name,
    toRider: target.name,
    movedQty,
    returnedQty,
    detail: detailParts.join("；")
  });
  if (returnedQty > 0) {
    ElMessage.warning(`交接完成：${target.name}接收${movedQty}件，${returnedQty}件装不下已退回候装区`);
  } else {
    ElMessage.success(`交接完成：${movedQty}件全部由${target.name}接管，泊位已释放`);
  }
}

function markBagClean(rider: Rider) {
  if (usedQty(rider.id) > 0) {
    ElMessage.warning("保温袋内还有货物，清空后才能清洗");
    return;
  }
  rider.bagClean = true;
  ElMessage.success(`${rider.name}的保温袋已清洗/更换，恢复未污染`);
}

function removeRider(rider: Rider) {
  if (state.orders.some((o) => o.riderId === rider.id)) {
    ElMessage.warning("该骑手名下还有订单记录，不能删除");
    return;
  }
  state.riders = state.riders.filter((r) => r.id !== rider.id);
  ElMessage.info("骑手已删除");
}

function addOrder() {
  if (!orderForm.batch.trim() || !orderForm.community.trim()) {
    ElMessage.warning("请填写批次号和社区名");
    return;
  }
  if (!Number.isFinite(orderForm.qty) || orderForm.qty < 1) {
    ElMessage.warning("件数至少为1");
    return;
  }
  state.orders.unshift({
    id: `ord-${Date.now().toString(36)}-${Math.random().toString(36).slice(2, 7)}`,
    batch: orderForm.batch.trim(),
    community: orderForm.community.trim(),
    zone: orderForm.zone,
    slot: orderForm.slot,
    qty: Math.floor(orderForm.qty),
    status: "候装",
    riderId: null,
    holdReason: "",
    createdAt: new Date().toISOString()
  });
  orderForm.batch = "";
  orderForm.community = "";
  ElMessage.success("社区订单已登记，进入候装区");
}

function addRider() {
  if (!riderForm.name.trim()) {
    ElMessage.warning("请填写骑手姓名");
    return;
  }
  if (!Number.isFinite(riderForm.capacity) || riderForm.capacity < 1) {
    ElMessage.warning("容量至少为1件");
    return;
  }
  if (!riderForm.berth.trim()) {
    ElMessage.warning("请填写泊位号");
    return;
  }
  state.riders.push({
    id: `rid-${Date.now().toString(36)}-${Math.random().toString(36).slice(2, 7)}`,
    name: riderForm.name.trim(),
    shift: riderForm.shift,
    bagClean: riderForm.bagClean,
    capacity: Math.floor(riderForm.capacity),
    berth: riderForm.berth.trim(),
    state: "在岗"
  });
  riderForm.name = "";
  riderForm.berth = "";
  ElMessage.success("骑手已排班上岗");
}

async function resetDay() {
  try {
    await ElMessageBox.confirm("将清空当天装车/交接数据并重新生成演示数据，确定继续？", "重置当日数据", {
      confirmButtonText: "重置",
      cancelButtonText: "取消",
      type: "warning"
    });
  } catch {
    return;
  }
  const fresh = seedState();
  state.date = fresh.date;
  state.orders = fresh.orders;
  state.riders = fresh.riders;
  state.handovers = fresh.handovers;
  Object.keys(targets).forEach((k) => delete targets[k]);
  Object.keys(takeovers).forEach((k) => delete takeovers[k]);
  ElMessage.success("当日数据已重置");
}

const reconcileRows = computed(() =>
  state.riders.map((r) => {
    const loaded = riderLoadedOrders(r.id);
    const departed = state.orders.filter((o) => o.riderId === r.id && o.status === "已出发");
    const sum = (list: CommunityOrder[]) => list.reduce((s, o) => s + o.qty, 0);
    return {
      rider: r,
      loadedQty: sum(loaded),
      loadedCount: loaded.length,
      departedQty: sum(departed),
      departedCount: departed.length
    };
  })
);

const heldOrders = computed(() => waitingOrders.value.filter((o) => o.holdReason));
</script>

<template>
  <main class="app">
    <div class="shell">
      <header class="topbar">
        <div>
          <p class="eyebrow">生鲜团购 · 末端配送装车台</p>
          <h1>晚间生鲜装车台</h1>
          <p class="subtitle">
            社区订单按批次、温区、承诺时段到站登记；骑手按班次排班，绑定保温袋与容量。
            冷藏件只能装入未污染保温袋，装不下的订单留在候装区并写明原因；骑手请假由同班次在岗人员接管未出发订单，原泊位当天释放。
          </p>
        </div>
        <div class="stack">
          <span class="tag">Vue3</span>
          <span class="tag">TypeScript</span>
          <span class="tag">Element Plus</span>
          <span class="tag">localStorage</span>
        </div>
      </header>

      <section class="metrics">
        <article v-for="item in metrics" :key="item.label" class="metric">
          <span>{{ item.label }}</span>
          <strong>{{ item.value }}</strong>
        </article>
      </section>

      <nav class="tabs">
        <button :class="{ active: tab === 'dock' }" class="tab" type="button" @click="tab = 'dock'">装车作业</button>
        <button :class="{ active: tab === 'riders' }" class="tab" type="button" @click="tab = 'riders'">骑手与泊位</button>
        <button :class="{ active: tab === 'handovers' }" class="tab" type="button" @click="tab = 'handovers'">
          交接记录<span v-if="state.handovers.length" class="tab-badge">{{ state.handovers.length }}</span>
        </button>
        <button :class="{ active: tab === 'audit' }" class="tab" type="button" @click="tab = 'audit'">当日核对</button>
        <span class="today">作业日 {{ state.date }} · 共 {{ batchCount }} 个批次</span>
      </nav>

      <!-- 装车作业 -->
      <section v-show="tab === 'dock'" class="workspace">
        <form class="panel" @submit.prevent="addOrder">
          <h2>社区订单登记</h2>
          <div class="form-grid">
            <label>
              批次号
              <input v-model="orderForm.batch" placeholder="如 T0926-晚团1" required />
            </label>
            <label>
              社区
              <input v-model="orderForm.community" placeholder="如 阳光花园" required />
            </label>
            <label>
              温区
              <select v-model="orderForm.zone">
                <option v-for="zone in ZONES" :key="zone" :value="zone">{{ zone }}</option>
              </select>
            </label>
            <label>
              承诺时段
              <select v-model="orderForm.slot">
                <option v-for="slot in SLOTS" :key="slot" :value="slot">{{ slot }}</option>
              </select>
            </label>
            <label>
              件数
              <input v-model.number="orderForm.qty" type="number" min="1" required />
            </label>
            <button type="submit">登记入候装区</button>
            <p class="form-hint">登记后进入候装区；装车时校验温区混装、保温袋污染与容量，装不下将写明原因留在候装区。</p>
          </div>
        </form>

        <div class="board">
          <section class="list-panel">
            <div class="toolbar">
              <h2>候装区 <em>{{ waitingOrders.length }} 单</em></h2>
              <button type="button" @click="autoLoad">智能配载</button>
            </div>
            <div class="record-grid">
              <div v-if="waitingOrders.length === 0" class="empty">候装区已清空</div>
              <article v-for="order in waitingOrders" :key="order.id" class="record order-card">
                <div class="record-head">
                  <p class="record-title">{{ order.batch }} · {{ order.community }}</p>
                  <span class="zone-tag" :class="order.zone === '冷藏' ? 'cold' : 'ambient'">{{ order.zone }}</span>
                </div>
                <div class="details">
                  <span>承诺时段：{{ order.slot }}</span>
                  <span>数量：{{ order.qty }} 件</span>
                </div>
                <div v-if="order.holdReason" class="warn">候装原因：{{ order.holdReason }}</div>
                <div class="load-line">
                  <select v-model="targets[order.id]">
                    <option value="">选择装袋骑手</option>
                    <option v-for="r in onDutyRiders" :key="r.id" :value="r.id">
                      {{ r.name }}（{{ r.shift }} · 余{{ r.capacity - usedQty(r.id) }}件 ·
                      {{ r.bagClean ? '袋净' : '袋污' }}{{ loadedZone(r.id) ? ' · 已装' + loadedZone(r.id) : '' }}）
                    </option>
                  </select>
                  <button type="button" :disabled="!targets[order.id]" @click="loadOrder(order)">装车</button>
                  <button class="danger" type="button" @click="removeOrder(order)">删除</button>
                </div>
              </article>
            </div>
          </section>

          <section class="list-panel">
            <div class="toolbar">
              <h2>已装待发 <em>{{ loadedOrders.length }} 单</em></h2>
            </div>
            <div v-if="activeLoadedRiders.length === 0" class="empty">暂无已装订单</div>
            <div v-for="r in activeLoadedRiders" :key="r.id" class="rider-load">
              <div class="rider-load-head">
                <strong>{{ r.name }}</strong>
                <span class="muted">{{ r.shift }} · 泊位 {{ r.berth }} · 保温袋{{ r.bagClean ? '未污染' : '已污染' }}</span>
                <span class="cap-text">{{ usedQty(r.id) }}/{{ r.capacity }} 件</span>
              </div>
              <div class="bar-track slim">
                <div class="bar-fill" :class="{ full: usedQty(r.id) >= r.capacity }" :style="{ width: `${capacityPercent(r)}%` }" />
              </div>
              <div class="chip-row">
                <span v-for="o in riderLoadedOrders(r.id)" :key="o.id" class="chip">
                  {{ o.batch }}·{{ o.community }}
                  <i class="zone-dot" :class="o.zone === '冷藏' ? 'cold' : 'ambient'" />
                  {{ o.zone }} {{ o.qty }}件
                  <button class="link-btn" type="button" @click="unloadOrder(o)">卸回</button>
                </span>
              </div>
            </div>
          </section>

          <section class="list-panel">
            <div class="toolbar">
              <h2>已出发 <em>{{ departedOrders.length }} 单</em></h2>
            </div>
            <div v-if="departedOrders.length === 0" class="empty">尚无骑手出发</div>
            <div class="chip-row">
              <span v-for="o in departedOrders" :key="o.id" class="chip departed">
                {{ riderName(o.riderId) }} · {{ o.batch }}·{{ o.community }}
                <i class="zone-dot" :class="o.zone === '冷藏' ? 'cold' : 'ambient'" />
                {{ o.zone }} {{ o.qty }}件 · {{ o.slot }}
              </span>
            </div>
          </section>
        </div>
      </section>

      <!-- 骑手与泊位 -->
      <section v-show="tab === 'riders'" class="workspace">
        <form class="panel" @submit.prevent="addRider">
          <h2>骑手排班登记</h2>
          <div class="form-grid">
            <label>
              姓名
              <input v-model="riderForm.name" placeholder="如 李明" required />
            </label>
            <label>
              班次
              <select v-model="riderForm.shift">
                <option v-for="shift in SHIFTS" :key="shift" :value="shift">{{ shift }}</option>
              </select>
            </label>
            <label>
              保温袋状态
              <select v-model="riderForm.bagClean">
                <option :value="true">未污染（可装冷藏件）</option>
                <option :value="false">已污染（仅限常温件）</option>
              </select>
            </label>
            <label>
              容量（件）
              <input v-model.number="riderForm.capacity" type="number" min="1" required />
            </label>
            <label>
              泊位号
              <input v-model="riderForm.berth" placeholder="如 B-05" required />
            </label>
            <button type="submit">排班上岗</button>
            <p class="form-hint">保温袋装入常温件后标记为已污染，需清洗或更换后才能再装冷藏件。</p>
          </div>
        </form>

        <section class="list-panel">
          <div class="toolbar">
            <h2>骑手与泊位 <em>{{ state.riders.length }} 人</em></h2>
          </div>
          <div class="rider-grid">
            <article v-for="r in state.riders" :key="r.id" class="rider-card" :class="r.state">
              <div class="record-head">
                <p class="record-title">{{ r.name }}</p>
                <span class="state-badge" :class="r.state">{{ r.state }}</span>
              </div>
              <div class="details">
                <span>班次：{{ r.shift }}</span>
                <span>泊位：{{ r.berth || '当天已释放' }}</span>
                <span>
                  保温袋：
                  <em :class="r.bagClean ? 'bag-clean' : 'bag-dirty'">{{ r.bagClean ? '未污染' : '已污染' }}</em>
                </span>
                <span>
                  容量：{{ usedQty(r.id) }}/{{ r.capacity }} 件
                  <em v-if="loadedZone(r.id)">（已装{{ loadedZone(r.id) }}）</em>
                </span>
              </div>
              <div class="bar-track slim">
                <div class="bar-fill" :class="{ full: usedQty(r.id) >= r.capacity }" :style="{ width: `${capacityPercent(r)}%` }" />
              </div>

              <div v-if="riderLoadedOrders(r.id).length" class="chip-row">
                <span v-for="o in riderLoadedOrders(r.id)" :key="o.id" class="chip">
                  {{ o.batch }}·{{ o.community }}
                  <i class="zone-dot" :class="o.zone === '冷藏' ? 'cold' : 'ambient'" />
                  {{ o.qty }}件
                </span>
              </div>

              <div v-if="r.state === '在岗'" class="actions rider-actions">
                <button type="button" :disabled="usedQty(r.id) === 0" @click="depart(r)">出发</button>
                <button class="secondary" type="button" :disabled="usedQty(r.id) > 0" @click="markBagClean(r)">
                  标记已清洗
                </button>
              </div>
              <div v-if="r.state === '在岗' && shiftCandidates(r).length > 0" class="takeover-line">
                <select v-model="takeovers[r.id]">
                  <option value="">请假：选择同班次接管人</option>
                  <option v-for="t in shiftCandidates(r)" :key="t.id" :value="t.id">
                    {{ t.name }}（{{ t.shift }} · 余{{ t.capacity - usedQty(t.id) }}件）
                  </option>
                </select>
                <button class="danger" type="button" :disabled="!takeovers[r.id]" @click="takeLeave(r)">请假交接</button>
              </div>
              <p v-else-if="r.state === '在岗'" class="form-hint">同班次暂无其他在岗骑手可接管，不能请假交接。</p>
              <p v-if="r.state === '请假'" class="warn">已请假：未出发订单已交接，泊位当天释放。</p>
              <button v-if="r.state !== '在岗'" class="link-btn danger-text" type="button" @click="removeRider(r)">删除骑手档案</button>
            </article>
          </div>
        </section>
      </section>

      <!-- 交接记录 -->
      <section v-show="tab === 'handovers'" class="list-panel standalone">
        <div class="toolbar">
          <h2>交接记录 <em>{{ state.handovers.length }} 条</em></h2>
        </div>
        <div v-if="state.handovers.length === 0" class="empty">暂无交接记录；骑手请假交接后在此留痕。</div>
        <ol v-else class="timeline">
          <li v-for="h in state.handovers" :key="h.id" class="timeline-item">
            <div class="timeline-head">
              <span class="handover-arrow">
                <strong>{{ h.fromRider }}</strong> → <strong>{{ h.toRider }}</strong>
              </span>
              <span class="muted">{{ h.time }}</span>
            </div>
            <p class="handover-detail">{{ h.detail }}</p>
            <p class="handover-qty">
              交接已装数量：<b>{{ h.movedQty }}</b> 件；退回候装区：<b :class="h.returnedQty > 0 ? 'bag-dirty' : ''">{{ h.returnedQty }}</b> 件
            </p>
          </li>
        </ol>
      </section>

      <!-- 当日核对 -->
      <section v-show="tab === 'audit'" class="audit">
        <section class="list-panel">
          <div class="toolbar">
            <h2>当日核对 · {{ state.date }}</h2>
            <button class="secondary" type="button" @click="resetDay">重置演示数据</button>
          </div>
          <p class="form-hint">数据保存在本机浏览器，当天重新打开页面可继续核对装车与转单结果；跨天自动开新作业日。</p>
          <div class="audit-stat-grid">
            <div class="audit-stat"><span>批次</span><strong>{{ batchCount }}</strong></div>
            <div class="audit-stat"><span>订单</span><strong>{{ state.orders.length }} 单</strong></div>
            <div class="audit-stat"><span>候装滞留</span><strong>{{ heldOrders.length }} 单</strong></div>
            <div class="audit-stat"><span>转单交接</span><strong>{{ state.handovers.length }} 次</strong></div>
          </div>
        </section>

        <section class="list-panel">
          <div class="toolbar"><h2>装车结果（按骑手）</h2></div>
          <div class="table-wrap">
            <table class="audit-table">
              <thead>
                <tr><th>骑手</th><th>班次</th><th>状态</th><th>泊位</th><th>已装待发</th><th>已出发</th></tr>
              </thead>
              <tbody>
                <tr v-for="row in reconcileRows" :key="row.rider.id">
                  <td>{{ row.rider.name }}</td>
                  <td>{{ row.rider.shift }}</td>
                  <td><span class="state-badge" :class="row.rider.state">{{ row.rider.state }}</span></td>
                  <td>{{ row.rider.berth || '已释放' }}</td>
                  <td>{{ row.loadedCount }} 单 / {{ row.loadedQty }} 件</td>
                  <td>{{ row.departedCount }} 单 / {{ row.departedQty }} 件</td>
                </tr>
              </tbody>
            </table>
          </div>
        </section>

        <section class="list-panel">
          <div class="toolbar"><h2>候装滞留与转单结果</h2></div>
          <h3 class="sub-title">候装区（装不下 / 拒收均写明原因）</h3>
          <div v-if="waitingOrders.length === 0" class="empty">无滞留</div>
          <ul v-else class="audit-list">
            <li v-for="o in waitingOrders" :key="o.id">
              <i class="zone-dot" :class="o.zone === '冷藏' ? 'cold' : 'ambient'" />
              {{ o.batch }} · {{ o.community }} · {{ o.zone }} · {{ o.qty }}件 · {{ o.slot }}
              <em :class="o.holdReason ? 'bag-dirty' : 'muted'">{{ o.holdReason || '待配载' }}</em>
            </li>
          </ul>

          <h3 class="sub-title">转单（请假交接）结果</h3>
          <div v-if="state.handovers.length === 0" class="empty">当日暂无转单</div>
          <ul v-else class="audit-list">
            <li v-for="h in state.handovers" :key="h.id">
              {{ h.time }}｜{{ h.fromRider }} → {{ h.toRider }}｜交接 {{ h.movedQty }} 件，退回 {{ h.returnedQty }} 件
              <em class="muted">{{ h.detail }}</em>
            </li>
          </ul>
        </section>
      </section>
    </div>
  </main>
</template>
