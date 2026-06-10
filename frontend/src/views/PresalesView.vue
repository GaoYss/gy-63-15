<template>
  <div class="page-stack">
    <section class="stats-grid">
      <StatCard label="活动数量" :value="presales.length" hint="当前预售批次" :icon="CalendarClock" />
      <StatCard label="总名额" :value="totalQuota" hint="可预约库存" :icon="Package" />
      <StatCard label="已预约" :value="totalReserved" hint="会员预订份数" :icon="ShoppingBag" />
      <StatCard label="订单数" :value="orders.length" hint="预售订单" :icon="ClipboardList" />
    </section>

  <div class="content-grid two-columns">
    <section class="panel">
      <div class="panel-header">
        <h2>预售活动</h2>
      </div>
      <div class="card-grid">
        <article v-for="presale in presales" :key="presale.id" class="presale-card">
          <div>
            <strong>{{ presale.title }}</strong>
            <span>{{ presale.start_date }} 至 {{ presale.end_date }}</span>
          </div>
          <div class="price-row">
            <b>￥{{ presale.price }}</b>
            <span>原价 ￥{{ presale.original_price }}</span>
          </div>
          <div class="progress-track" :class="progressClass(presale)">
            <div :style="{ width: `${reservedPercent(presale)}%` }"></div>
          </div>
          <small>剩余 {{ getRemaining(presale) }} 名额，已预约 {{ presale.reserved }} / {{ presale.quota }}，{{ presale.pickup_date }} 提货</small>
          <button
            class="secondary-button"
            :class="buttonClass(presale)"
            :disabled="getRemaining(presale) <= 0 || reservingIds.has(presale.id)"
            @click="reserve(presale.id)"
          >{{ buttonText(presale) }}</button>
          <p v-if="cardFeedback[presale.id]" class="card-feedback" :class="`card-feedback--${cardFeedback[presale.id].type}`" role="alert">
            <span aria-hidden="true">{{ cardFeedback[presale.id].type === 'success' ? '✓' : '⚠' }}</span> {{ cardFeedback[presale.id].text }}
            <button type="button" class="card-feedback__close" @click="clearCardFeedback(presale.id)" aria-label="关闭提示">×</button>
          </p>
        </article>
      </div>
      <div class="order-list">
        <h3>预约订单</h3>
        <article v-for="order in orders" :key="order.id" class="compact-card">
          <div>
            <strong>订单 #{{ order.id }}</strong>
            <span>{{ memberName(order.member_id) }} · {{ presaleName(order.presale_id) }}</span>
          </div>
          <b>￥{{ order.amount }} / {{ order.quantity }}份</b>
        </article>
        <EmptyState v-if="!orders.length" text="暂无预约订单" />
      </div>
    </section>

    <section class="panel">
      <div class="panel-header">
        <h2>新增预售</h2>
      </div>
      <MessageBanner :message="message" :type="messageType" />
      <form class="form-stack" @submit.prevent="submit">
        <label>活动标题<input v-model="form.title" required /></label>
        <label>水果名称<input v-model="form.fruit_name" required /></label>
        <div class="form-row">
          <label>预售价<input v-model.number="form.price" min="0" step="0.1" type="number" /></label>
          <label>原价<input v-model.number="form.original_price" min="0" step="0.1" type="number" /></label>
        </div>
        <div class="form-row">
          <label>开始日期<input v-model="form.start_date" type="date" /></label>
          <label>结束日期<input v-model="form.end_date" type="date" /></label>
        </div>
        <label>提货日期<input v-model="form.pickup_date" type="date" /></label>
        <label>名额<input v-model.number="form.quota" min="1" type="number" /></label>
        <button class="primary-button" type="submit">发布活动</button>
      </form>
    </section>
  </div>
  </div>
</template>

<script setup>
import { computed, onMounted, reactive, ref } from 'vue'
import { CalendarClock, ClipboardList, Package, ShoppingBag } from 'lucide-vue-next'

import { fallbackMembers, fallbackPresales } from '../api/fallback'
import { memberApi } from '../api/members'
import { presaleApi } from '../api/presales'
import EmptyState from '../components/EmptyState.vue'
import MessageBanner from '../components/MessageBanner.vue'
import StatCard from '../components/StatCard.vue'
import { keepList } from '../utils/dataState'

const presales = ref([...fallbackPresales])
const members = ref([...fallbackMembers])
const orders = ref([])
const message = ref('')
const messageType = ref('success')
const reservingIds = ref(new Set())
const cardFeedback = reactive({})
const feedbackTimers = {}
const form = reactive({
  title: '',
  fruit_name: '',
  price: 39.9,
  original_price: 49.9,
  start_date: '2026-06-01',
  end_date: '2026-06-15',
  pickup_date: '2026-06-18',
  quota: 100,
  reserved: 0,
})

const totalQuota = computed(() => presales.value.reduce((sum, item) => sum + item.quota, 0))
const totalReserved = computed(() => presales.value.reduce((sum, item) => sum + item.reserved, 0))

function getRemaining(presale) {
  return presale.remaining ?? presale.quota - presale.reserved
}

function reservedPercent(presale) {
  return ((presale.quota - getRemaining(presale)) / presale.quota) * 100
}

function progressClass(presale) {
  const remaining = getRemaining(presale)
  if (remaining <= 0) return 'progress-track--full'
  if (remaining <= Math.ceil(presale.quota * 0.2)) return 'progress-track--low'
  return ''
}

function buttonClass(presale) {
  const remaining = getRemaining(presale)
  const isReserving = reservingIds.value.has(presale.id)
  return {
    'secondary-button--urgent': remaining > 0 && remaining <= Math.ceil(presale.quota * 0.2),
    'secondary-button--hot': remaining > 0 && remaining <= 5,
    'secondary-button--loading': isReserving,
  }
}

function buttonText(presale) {
  if (reservingIds.value.has(presale.id)) return '处理中…'
  if (getRemaining(presale) <= 0) return '名额已满'
  const remaining = getRemaining(presale)
  if (remaining <= 5) return `仅剩 ${remaining} 份，立即预约`
  return '预约一份'
}

async function loadData() {
  const [presaleList, memberList, orderList] = await Promise.all([
    presaleApi.list().catch(() => fallbackPresales),
    memberApi.list().catch(() => fallbackMembers),
    presaleApi.orders().catch(() => []),
  ])
  presales.value = keepList(presaleList, presales.value)
  members.value = keepList(memberList, members.value)
  orders.value = Array.isArray(orderList) ? orderList : orders.value
}

async function submit() {
  try {
    await presaleApi.create({ ...form })
    Object.assign(form, {
      title: '',
      fruit_name: '',
      price: 39.9,
      original_price: 49.9,
      start_date: '2026-06-01',
      end_date: '2026-06-15',
      pickup_date: '2026-06-18',
      quota: 100,
      reserved: 0,
    })
    message.value = '预售活动已发布'
    messageType.value = 'success'
    await loadData()
  } catch (error) {
    message.value = error.message
    messageType.value = 'error'
  }
}

function clearCardFeedback(presaleId) {
  delete cardFeedback[presaleId]
  if (feedbackTimers[presaleId]) {
    clearTimeout(feedbackTimers[presaleId])
    delete feedbackTimers[presaleId]
  }
}

function showCardFeedback(presaleId, type, text, duration) {
  cardFeedback[presaleId] = { type, text }
  if (feedbackTimers[presaleId]) {
    clearTimeout(feedbackTimers[presaleId])
  }
  feedbackTimers[presaleId] = setTimeout(() => {
    delete cardFeedback[presaleId]
    delete feedbackTimers[presaleId]
  }, duration)
}

async function reserve(presaleId) {
  if (reservingIds.value.has(presaleId)) return
  const presale = presales.value.find((p) => p.id === presaleId)
  if (!presale || getRemaining(presale) <= 0) return

  reservingIds.value = new Set(reservingIds.value)
  reservingIds.value.add(presaleId)
  clearCardFeedback(presaleId)

  const snapshot = {
    reserved: presale.reserved,
    remaining: 'remaining' in presale ? presale.remaining : undefined,
  }
  presale.reserved += 1
  if ('remaining' in presale) {
    presale.remaining -= 1
  }

  const tempOrderId = Date.now()
  const memberId = members.value[0]?.id || 1

  try {
    await presaleApi.reserve({ member_id: memberId, presale_id: presaleId, quantity: 1 })

    orders.value.unshift({
      id: tempOrderId,
      member_id: memberId,
      presale_id: presaleId,
      quantity: 1,
      amount: presale.price,
      status: 'reserved',
    })

    showCardFeedback(presaleId, 'success', '预约成功', 2500)
    message.value = '预约成功'
    messageType.value = 'success'
    await Promise.all([
      loadData(),
      new Promise((resolve) => setTimeout(resolve, 500)),
    ])
  } catch (error) {
    presale.reserved = snapshot.reserved
    if ('remaining' in presale) {
      presale.remaining = snapshot.remaining
    }
    showCardFeedback(presaleId, 'error', error.message || '预约失败，请重试', 4500)
    message.value = error.message
    messageType.value = 'error'
  } finally {
    reservingIds.value = new Set(reservingIds.value)
    reservingIds.value.delete(presaleId)
  }
}

function memberName(memberId) {
  return members.value.find((member) => member.id === memberId)?.name || `会员${memberId}`
}

function presaleName(presaleId) {
  return presales.value.find((presale) => presale.id === presaleId)?.title || `活动${presaleId}`
}

onMounted(loadData)
</script>
