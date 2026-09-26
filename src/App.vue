<script setup lang="ts">
import { computed, reactive, ref } from "vue";

type Zone = "冷藏" | "常温";
type OrderStatus = "候装" | "已装车" | "已出发";

interface Order {
  id: string;
  community: string;
  batch: string;
  zone: Zone;
  slot: string;
  units: number;
  status: OrderStatus;
  riderId: string | null;
  bagId: string | null;
  holdReason: string;
  notes: string;
  createdDate: string;
  createdAt: string;
  loadedDate: string | null;
}

interface Bag {
  id: string;
  label: string;
  capacity: number;
  contaminated: boolean;
}

interface Rider {
  id: string;
  name: string;
  shift: string;
  berth: string;
  onLeave: boolean;
  leaveDate: string | null;
  bags: Bag[];
}

interface Handover {
  id: string;
  orderLabel: string;
  fromRider: string;
  toRider: string;
  loadedUnits: number;
  date: string;
  time: string;
}

interface State {
  orders: Order[];
  riders: Rider[];
  handovers: Handover[];
}

const storageKey = "hxwlfront-15-loading-dock";
const ZONES: Zone[] = ["冷藏", "常温"];
const SHIFTS = ["早班", "中班", "晚班"];
const SLOTS = ["08:00-10:00", "10:00-12:00", "14:00-16:00", "16:00-18:00", "19:00-21:00"];
const STATUSES: OrderStatus[] = ["候装", "已装车", "已出发"];
const metricLabels = ["今日登记订单", "今日装车件数", "候装滞留", "今日交接"];

function todayStr() {
  const d = new Date();
  return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, "0")}-${String(d.getDate()).padStart(2, "0")}`;
}

function seedState(): State {
  return {
    riders: [
      {
        id: "rider-a",
        name: "骑手A",
        shift: "早班",
        berth: "1号泊位",
        onLeave: false,
        leaveDate: null,
        bags: [
          { id: "bag-a1", label: "冷藏袋A1", capacity: 20, contaminated: false },
          { id: "bag-a2", label: "保温袋A2", capacity: 15, contaminated: false }
        ]
      },
      {
        id: "rider-b",
        name: "骑手B",
        shift: "早班",
        berth: "2号泊位",
        onLeave: false,
        leaveDate: null,
        bags: [{ id: "bag-b1", label: "保温袋B1", capacity: 18, contaminated: true }]
      },
      {
        id: "rider-c",
        name: "骑手C",
        shift: "晚班",
        berth: "3号泊位",
        onLeave: false,
        leaveDate: null,
        bags: [{ id: "bag-c1", label: "冷藏袋C1", capacity: 12, contaminated: false }]
      }
    ],
    orders: [
      {
        id: "order-1",
        community: "世纪大道社区",
        batch: "批次0901",
        zone: "冷藏",
        slot: "10:00-12:00",
        units: 8,
        status: "候装",
        riderId: null,
        bagId: null,
        holdReason: "",
        notes: "到站优先装车",
        createdDate: todayStr(),
        createdAt: new Date().toISOString(),
        loadedDate: null
      },
      {
        id: "order-2",
        community: "陆家嘴社区",
        batch: "批次0902",
        zone: "常温",
        slot: "14:00-16:00",
        units: 6,
        status: "候装",
        riderId: null,
        bagId: null,
        holdReason: "",
        notes: "待确认",
        createdDate: todayStr(),
        createdAt: new Date().toISOString(),
        loadedDate: null
      },
      {
        id: "order-3",
        community: "联洋社区",
        batch: "批次0901",
        zone: "冷藏",
        slot: "08:00-10:00",
        units: 4,
        status: "已装车",
        riderId: "rider-a",
        bagId: "bag-a1",
        holdReason: "",
        notes: "已装冷藏袋A1",
        createdDate: todayStr(),
        createdAt: new Date().toISOString(),
        loadedDate: todayStr()
      }
    ],
    handovers: []
  };
}

function loadState(): State {
  const raw = localStorage.getItem(storageKey);
  if (raw) {
    try {
      return JSON.parse(raw) as State;
    } catch {
      // fall through to seed
    }
  }
  return seedState();
}

const state = loadState();
const orders = ref<Order[]>(state.orders);
const riders = ref<Rider[]>(state.riders);
const handovers = ref<Handover[]>(state.handovers);

const orderForm = reactive({ community: "", batch: "", zone: "冷藏" as Zone, slot: SLOTS[1], units: 1, notes: "" });
const riderForm = reactive({ name: "", shift: SHIFTS[0], berth: "" });
const bagDrafts = reactive<Record<string, { label: string; capacity: number }>>({});
const assignDrafts = reactive<Record<string, string>>({});
const transferDrafts = reactive<Record<string, string>>({});
const zoneFilter = ref("全部温区");

function ensureBagDraft(riderId: string) {
  if (!bagDrafts[riderId]) bagDrafts[riderId] = { label: "", capacity: 10 };
}
riders.value.forEach((rider) => ensureBagDraft(rider.id));

function persist() {
  const snapshot: State = { orders: orders.value, riders: riders.value, handovers: handovers.value };
  localStorage.setItem(storageKey, JSON.stringify(snapshot));
}

function riderOf(order: Order) {
  return riders.value.find((rider) => rider.id === order.riderId) || null;
}

function riderName(id: string | null) {
  return riders.value.find((rider) => rider.id === id)?.name || "未分配";
}

function bagLabel(id: string | null) {
  if (!id) return "—";
  for (const rider of riders.value) {
    const bag = rider.bags.find((item) => item.id === id);
    if (bag) return bag.label;
  }
  return "—";
}

function bagUsed(bagId: string) {
  return orders.value
    .filter((order) => order.bagId === bagId && order.status !== "候装")
    .reduce((sum, order) => sum + order.units, 0);
}

const activeRiders = computed(() => riders.value.filter((rider) => !rider.onLeave));

const stagingOrders = computed(() =>
  orders.value.filter((order) => {
    if (order.status !== "候装") return false;
    if (zoneFilter.value === "全部温区") return true;
    return order.zone === zoneFilter.value;
  })
);

const loadedOrders = computed(() => orders.value.filter((order) => order.status !== "候装"));

const todayHandovers = computed(() => handovers.value.filter((item) => item.date === todayStr()));

const metrics = computed(() => [
  orders.value.filter((order) => order.createdDate === todayStr()).length,
  orders.value
    .filter((order) => order.status !== "候装" && order.loadedDate === todayStr())
    .reduce((sum, order) => sum + order.units, 0),
  orders.value.filter((order) => order.status === "候装").length,
  todayHandovers.value.length
]);

const chartRows = computed(() =>
  STATUSES.map((status) => ({
    status,
    value: orders.value.filter((order) => order.status === status).length
  }))
);

const maxChart = computed(() => Math.max(1, ...chartRows.value.map((row) => row.value)));

function loadOrder(order: Order, riderId: string) {
  const rider = riders.value.find((item) => item.id === riderId);
  if (!rider) return;
  if (rider.onLeave) {
    order.holdReason = "该骑手已请假，无法装车，请交接给同班次骑手";
    persist();
    return;
  }
  // 冷藏件只能放未污染的袋子，常温件不限制
  const usable = rider.bags.filter((bag) => order.zone === "常温" || !bag.contaminated);
  if (usable.length === 0) {
    order.holdReason =
      rider.bags.length === 0 ? "该骑手尚未登记保温袋" : "冷藏件只能装未污染的保温袋，当前袋子均已污染";
    persist();
    return;
  }
  const bag = usable.find((item) => item.capacity - bagUsed(item.id) >= order.units);
  if (!bag) {
    order.holdReason = `可用保温袋剩余容量不足（需${order.units}件），留在候装区`;
    persist();
    return;
  }
  order.status = "已装车";
  order.riderId = rider.id;
  order.bagId = bag.id;
  order.holdReason = "";
  order.loadedDate = todayStr();
  persist();
}

function assignAndLoad(order: Order) {
  const riderId = assignDrafts[order.id];
  if (!riderId) return;
  loadOrder(order, riderId);
}

function depart(order: Order) {
  order.status = "已出发";
  persist();
}

function toggleLeave(rider: Rider) {
  rider.onLeave = !rider.onLeave;
  // 请假当天释放原泊位，返岗后恢复
  rider.leaveDate = rider.onLeave ? todayStr() : null;
  persist();
}

function sameShiftActive(order: Order) {
  const from = riderOf(order);
  if (!from) return [];
  return riders.value.filter((rider) => rider.id !== from.id && rider.shift === from.shift && !rider.onLeave);
}

function transferable(order: Order) {
  const rider = riderOf(order);
  return !!rider && rider.onLeave && order.status !== "已出发";
}

function transfer(order: Order) {
  const from = riderOf(order);
  const to = riders.value.find((rider) => rider.id === transferDrafts[order.id]);
  if (!from || !to || to.onLeave || to.shift !== from.shift) return;
  // 交接记下前后责任人和已装数量
  handovers.value.unshift({
    id: crypto.randomUUID(),
    orderLabel: `${order.community} / ${order.batch}`,
    fromRider: from.name,
    toRider: to.name,
    loadedUnits: order.status === "已装车" ? order.units : 0,
    date: todayStr(),
    time: new Date().toLocaleString("zh-CN", { hour12: false })
  });
  order.riderId = to.id;
  order.bagId = null;
  if (order.status === "已装车") {
    // 换骑手后重新按规则装车，装不下回候装区并写明原因
    order.status = "候装";
    order.loadedDate = null;
    loadOrder(order, to.id);
  }
  transferDrafts[order.id] = "";
  persist();
}

function addOrder() {
  orders.value.unshift({
    id: crypto.randomUUID(),
    community: orderForm.community,
    batch: orderForm.batch,
    zone: orderForm.zone,
    slot: orderForm.slot,
    units: Number(orderForm.units) || 1,
    status: "候装",
    riderId: null,
    bagId: null,
    holdReason: "",
    notes: orderForm.notes || "暂无备注",
    createdDate: todayStr(),
    createdAt: new Date().toISOString(),
    loadedDate: null
  });
  Object.assign(orderForm, { community: "", batch: "", zone: "冷藏", slot: SLOTS[1], units: 1, notes: "" });
  persist();
}

function addRider() {
  const rider: Rider = {
    id: crypto.randomUUID(),
    name: riderForm.name,
    shift: riderForm.shift,
    berth: riderForm.berth,
    onLeave: false,
    leaveDate: null,
    bags: []
  };
  riders.value.push(rider);
  ensureBagDraft(rider.id);
  Object.assign(riderForm, { name: "", shift: SHIFTS[0], berth: "" });
  persist();
}

function addBag(rider: Rider) {
  const draft = bagDrafts[rider.id];
  if (!draft || !draft.label || draft.capacity < 1) return;
  rider.bags.push({ id: crypto.randomUUID(), label: draft.label, capacity: Number(draft.capacity), contaminated: false });
  bagDrafts[rider.id] = { label: "", capacity: 10 };
  persist();
}

function toggleContaminated(bag: Bag) {
  bag.contaminated = !bag.contaminated;
  persist();
}

function removeOrder(id: string) {
  orders.value = orders.value.filter((order) => order.id !== id);
  persist();
}
</script>

<template>
  <main class="app">
    <div class="shell">
      <header class="topbar">
        <div>
          <p class="eyebrow">物流行业前端最小闭环</p>
          <h1>社区团购装车台</h1>
          <p class="subtitle">
            晚间生鲜团购到站后统一装车：订单按批次、温区和承诺时段登记，骑手按班次管理保温袋。
            冷藏件只装未污染的袋子，装不下留在候装区并写明原因；骑手请假后同班次可接管未出发订单，泊位当天释放。
          </p>
        </div>
        <div class="stack">
          <span class="tag">Vue3</span>
          <span class="tag">Vite</span>
          <span class="tag">TypeScript</span>
          <span class="tag">Element Plus</span>
        </div>
      </header>

      <section class="metrics">
        <article v-for="(label, index) in metricLabels" :key="label" class="metric">
          <span>{{ label }}</span>
          <strong>{{ metrics[index] }}</strong>
        </article>
      </section>

      <section class="workspace">
        <div class="side">
          <form class="panel" @submit.prevent="addOrder">
            <h2>登记社区订单</h2>
            <div class="form-grid">
              <label>
                社区
                <input v-model="orderForm.community" placeholder="如：世纪大道社区" required />
              </label>
              <label>
                批次
                <input v-model="orderForm.batch" placeholder="如：批次0901" required />
              </label>
              <label>
                温区
                <select v-model="orderForm.zone">
                  <option v-for="zone in ZONES" :key="zone">{{ zone }}</option>
                </select>
              </label>
              <label>
                承诺时段
                <select v-model="orderForm.slot">
                  <option v-for="slot in SLOTS" :key="slot">{{ slot }}</option>
                </select>
              </label>
              <label>
                件数
                <input v-model.number="orderForm.units" type="number" min="1" required />
              </label>
              <label>
                备注
                <textarea v-model="orderForm.notes" placeholder="填写团购到站或保鲜要求" />
              </label>
              <button type="submit">送入候装区</button>
            </div>
          </form>

          <section class="panel">
            <h2>骑手与保温袋</h2>
            <form class="inline-form" @submit.prevent="addRider">
              <input v-model="riderForm.name" placeholder="骑手姓名" required />
              <select v-model="riderForm.shift">
                <option v-for="shift in SHIFTS" :key="shift">{{ shift }}</option>
              </select>
              <input v-model="riderForm.berth" placeholder="泊位" required />
              <button type="submit">新增骑手</button>
            </form>

            <div class="record-grid">
              <article v-for="rider in riders" :key="rider.id" class="record">
                <div class="record-head">
                  <p class="record-title">{{ rider.name }} · {{ rider.shift }}</p>
                  <span class="status" :class="{ leave: rider.onLeave }">{{ rider.onLeave ? "请假中" : "在岗" }}</span>
                </div>
                <p class="berth">
                  泊位：{{ rider.berth }}
                  <em v-if="rider.onLeave">（{{ rider.leaveDate }} 请假，当天已释放）</em>
                </p>
                <div class="bag-list">
                  <div v-for="bag in rider.bags" :key="bag.id" class="bag-row">
                    <span>{{ bag.label }} · 容量{{ bag.capacity }} · 已装{{ bagUsed(bag.id) }}</span>
                    <span class="tag" :class="{ dirty: bag.contaminated }">{{ bag.contaminated ? "已污染" : "未污染" }}</span>
                    <button class="secondary" type="button" @click="toggleContaminated(bag)">
                      {{ bag.contaminated ? "消毒完成" : "标记污染" }}
                    </button>
                  </div>
                  <div v-if="rider.bags.length === 0" class="empty">尚未登记保温袋</div>
                </div>
                <form class="inline-form" @submit.prevent="addBag(rider)">
                  <input v-model="bagDrafts[rider.id].label" placeholder="保温袋名称" required />
                  <input v-model.number="bagDrafts[rider.id].capacity" type="number" min="1" placeholder="容量" required />
                  <button type="submit">添加袋子</button>
                </form>
                <div class="actions">
                  <button :class="rider.onLeave ? '' : 'danger'" type="button" @click="toggleLeave(rider)">
                    {{ rider.onLeave ? "返岗" : "请假" }}
                  </button>
                </div>
              </article>
            </div>
          </section>
        </div>

        <div class="side">
          <section class="list-panel">
            <div class="toolbar">
              <h2>候装区</h2>
              <select v-model="zoneFilter">
                <option>全部温区</option>
                <option v-for="zone in ZONES" :key="zone">{{ zone }}</option>
              </select>
            </div>
            <div class="record-grid">
              <div v-if="stagingOrders.length === 0" class="empty">候装区暂无订单</div>
              <article v-for="order in stagingOrders" :key="order.id" class="record">
                <div class="record-head">
                  <p class="record-title">{{ order.community }} / {{ order.batch }}</p>
                  <span class="status" :class="{ cold: order.zone === '冷藏' }">{{ order.zone }} · {{ order.status }}</span>
                </div>
                <div class="details">
                  <span>承诺时段: {{ order.slot }}</span>
                  <span>件数: {{ order.units }}</span>
                  <span>登记日期: {{ order.createdDate }}</span>
                  <span>当前骑手: {{ riderName(order.riderId) }}</span>
                </div>
                <p v-if="order.holdReason" class="note warn">滞留原因：{{ order.holdReason }}</p>
                <p class="note">{{ order.notes }}</p>
                <div v-if="transferable(order)" class="transfer-row">
                  <select v-model="transferDrafts[order.id]">
                    <option value="">选择同班次在岗骑手</option>
                    <option v-for="rider in sameShiftActive(order)" :key="rider.id" :value="rider.id">
                      {{ rider.name }}（{{ rider.shift }}）
                    </option>
                  </select>
                  <button type="button" :disabled="!transferDrafts[order.id]" @click="transfer(order)">交接接管</button>
                </div>
                <div class="actions">
                  <select v-model="assignDrafts[order.id]">
                    <option value="">选择在岗骑手</option>
                    <option v-for="rider in activeRiders" :key="rider.id" :value="rider.id">
                      {{ rider.name }}（{{ rider.shift }}）
                    </option>
                  </select>
                  <button type="button" :disabled="!assignDrafts[order.id]" @click="assignAndLoad(order)">装车</button>
                  <button class="danger" type="button" @click="removeOrder(order.id)">删除</button>
                </div>
              </article>
            </div>
          </section>

          <section class="list-panel">
            <div class="toolbar">
              <h2>装车清单</h2>
            </div>
            <div class="record-grid">
              <div v-if="loadedOrders.length === 0" class="empty">暂无已装车订单</div>
              <article v-for="order in loadedOrders" :key="order.id" class="record">
                <div class="record-head">
                  <p class="record-title">{{ order.community }} / {{ order.batch }}</p>
                  <span class="status" :class="{ cold: order.zone === '冷藏' }">{{ order.zone }} · {{ order.status }}</span>
                </div>
                <div class="details">
                  <span>承诺时段: {{ order.slot }}</span>
                  <span>件数: {{ order.units }}</span>
                  <span>骑手: {{ riderName(order.riderId) }}</span>
                  <span>保温袋: {{ bagLabel(order.bagId) }}</span>
                </div>
                <div v-if="transferable(order)" class="transfer-row">
                  <span class="leave-tip">骑手已请假，可交接给同班次在岗骑手：</span>
                  <select v-model="transferDrafts[order.id]">
                    <option value="">选择接管骑手</option>
                    <option v-for="rider in sameShiftActive(order)" :key="rider.id" :value="rider.id">
                      {{ rider.name }}（{{ rider.shift }}）
                    </option>
                  </select>
                  <button type="button" :disabled="!transferDrafts[order.id]" @click="transfer(order)">交接接管</button>
                </div>
                <div class="actions">
                  <button v-if="order.status === '已装车'" type="button" @click="depart(order)">出发配送</button>
                  <button class="danger" type="button" @click="removeOrder(order.id)">删除</button>
                </div>
              </article>
            </div>
          </section>

          <section class="list-panel">
            <div class="toolbar">
              <h2>当日核对 · {{ todayStr() }}</h2>
            </div>
            <div class="chips">
              <span class="tag">今日装车 {{ metrics[1] }} 件</span>
              <span class="tag">候装滞留 {{ metrics[2] }} 单</span>
              <span class="tag">今日交接 {{ metrics[3] }} 次</span>
            </div>
            <div class="record-grid">
              <div v-if="todayHandovers.length === 0" class="empty">今日暂无交接记录</div>
              <article v-for="item in todayHandovers" :key="item.id" class="record handover">
                <p class="record-title">{{ item.orderLabel }}</p>
                <div class="details">
                  <span>前责任人: {{ item.fromRider }}</span>
                  <span>接管人: {{ item.toRider }}</span>
                  <span>已装数量: {{ item.loadedUnits }} 件</span>
                  <span>时间: {{ item.time }}</span>
                </div>
              </article>
            </div>
            <div class="mini-chart">
              <div v-for="row in chartRows" :key="row.status" class="bar">
                <span>{{ row.status }}</span>
                <div class="bar-track"><div class="bar-fill" :style="{ width: `${(row.value / maxChart) * 100}%` }" /></div>
                <strong>{{ row.value }}</strong>
              </div>
            </div>
          </section>
        </div>
      </section>
    </div>
  </main>
</template>
