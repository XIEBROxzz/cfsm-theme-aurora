<script setup>
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'
import { monitor } from '../stores/monitor'
import { t } from '../utils/i18n'
import { formatBytes, formatSpeed, monthTraffic, aggregateMonthlyTraffic, isServerOnline } from '../utils/format'
import AppIcon from './AppIcon.vue'

const stats = computed(() => monitor.stats)

const month = computed(() => {
  const summary = aggregateMonthlyTraffic(monitor.servers, monitor.now)
  return { ...summary, rx: summary.knownNodes ? summary.rx : null, tx: summary.knownNodes ? summary.tx : null, total: summary.knownNodes ? summary.total : null }
})

const speed = computed(() => {
  let inbound = 0
  let outbound = 0
  for (const server of monitor.servers.filter(server => isServerOnline(server, monitor.now))) {
    inbound += Number(server.net_in_speed) || 0
    outbound += Number(server.net_out_speed) || 0
  }
  return { inbound, outbound, total: inbound + outbound }
})

const busiest = computed(() => {
  let top = null
  for (const server of monitor.servers) {
    const traffic = monthTraffic(server, monitor.now)
    if (traffic === null) continue
    if (!top || traffic > top.traffic) top = { name: server.name || server.id, traffic }
  }
  return top && top.traffic > 0 ? top : null
})

// A short local ring buffer, sampled independently of the WebSocket flush, so
// the speed tile keeps the reference's sparkline shape without adding history
// bookkeeping to the store.
const history = ref([])
let historyTimer = null

function sampleSpeed() {
  const next = history.value.slice(-29)
  next.push({ rx: speed.value.inbound, tx: speed.value.outbound })
  history.value = next
}

onMounted(() => {
  sampleSpeed()
  historyTimer = setInterval(sampleSpeed, 2000)
})

onBeforeUnmount(() => {
  clearInterval(historyTimer)
})

const spark = computed(() => {
  const values = history.value
  if (values.length < 2) return null
  const top = Math.max(...values.flatMap((item) => [item.rx, item.tx]), 1)
  const width = values.length - 1
  const line = (key) => values
    .map((item, index) => `${(index / width) * 100},${19 - (item[key] / top) * 18}`)
    .join(' ')
  return { rx: line('rx'), tx: line('tx') }
})
</script>

<template>
  <section class="summary-grid" aria-label="Summary">
    <article class="summary-tile">
      <div class="tile-head">
        <span class="tile-label">
          <AppIcon name="server" :size="14" />
          {{ t('nodes') }}
        </span>
      </div>
      <div class="tile-value tnum">
        {{ stats.online }}<span class="tile-divider">/</span>{{ stats.total }}
      </div>
      <div class="tile-foot">
        <span v-if="stats.offline" class="tone-bad">{{ t('offlineCount', { n: stats.offline }) }}</span>
        <span v-else class="tone-good">{{ t('allOnline') }}</span>
      </div>
    </article>

    <article v-if="monitor.sysConfig.show_tf" class="summary-tile">
      <div class="tile-head">
        <span class="tile-label">
          <AppIcon name="activity" :size="14" />
          {{ t('busiestNode') }}
        </span>
      </div>
      <div class="tile-value tnum">{{ busiest ? formatBytes(busiest.traffic) : '—' }}</div>
      <div class="tile-foot tile-truncate">{{ busiest ? busiest.name : t('noTraffic') }}</div>
    </article>

    <article v-if="monitor.sysConfig.show_tf" class="summary-tile" data-monthly>
      <div class="tile-head">
        <span class="tile-label">
          <AppIcon name="arrowDownUp" :size="14" />
          {{ t('trafficCard') }}
        </span>
        <span class="tile-badge">UTC · {{ month.period.key }}</span>
      </div>
      <div class="tile-foot">{{ month.knownNodes }}/{{ month.totalNodes }} {{ t('nodes') }}<span v-if="!month.complete"> · {{ t('monthlyPartial') }}</span></div>
      <div class="tile-value tnum">{{ formatBytes(month.total) }}</div>
      <div class="tile-foot tile-split">
        <span>↓ {{ formatBytes(month.rx) }}</span>
        <span class="tile-sep">·</span>
        <span>↑ {{ formatBytes(month.tx) }}</span>
      </div>
    </article>

    <article class="summary-tile">
      <div class="tile-head">
        <span class="tile-label">
          <AppIcon name="bolt" :size="14" />
          {{ t('liveSpeed') }}
        </span>
      </div>
      <div class="tile-value tnum">{{ formatSpeed(speed.total) }}</div>
      <div class="tile-foot tile-speed">
        <div class="tile-split">
          <span>↓ {{ formatSpeed(speed.inbound) }}</span>
          <span class="tile-sep">·</span>
          <span>↑ {{ formatSpeed(speed.outbound) }}</span>
        </div>
        <svg v-if="spark && monitor.sparkline" class="tile-spark" viewBox="0 0 100 20" preserveAspectRatio="none" aria-hidden="true">
          <polyline :points="spark.rx" fill="none" stroke="currentColor" stroke-width="1.25" vector-effect="non-scaling-stroke" />
          <polyline :points="spark.tx" fill="none" stroke="currentColor" stroke-width="1.25" vector-effect="non-scaling-stroke" class="spark-muted" />
        </svg>
      </div>
    </article>
  </section>
</template>

<style scoped>
.summary-grid {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: var(--gap);
}

.summary-tile {
  position: relative;
  display: flex;
  min-width: 0;
  min-height: 110px;
  flex-direction: column;
  justify-content: space-between;
  overflow: hidden;
  padding: 14px;
  border: 1px solid var(--line);
  border-radius: var(--radius);
  background: var(--surface);
  box-shadow: var(--shadow-soft);
}

.summary-tile::before {
  position: absolute;
  top: 0;
  right: 0;
  left: 0;
  height: 2px;
  background: linear-gradient(90deg, #e11d48, #7c3aed 48%, #059669);
  content: '';
  opacity: 0.75;
}

.tile-head {
  display: flex;
  min-width: 0;
  align-items: center;
  justify-content: space-between;
  gap: 6px;
  color: var(--muted);
  font-size: 12px;
}

.tile-label {
  display: inline-flex;
  min-width: 0;
  align-items: center;
  gap: 6px;
  color: var(--text-soft);
  font-weight: 500;
  white-space: nowrap;
}

.tile-label :deep(svg) {
  flex: 0 0 auto;
  color: var(--muted);
}

.tile-badge {
  flex: 0 0 auto;
  padding: 2px 7px;
  border: 1px solid var(--line);
  border-radius: 999px;
  background: var(--bg-soft);
  color: var(--muted);
  font-size: 10px;
  line-height: 1.2;
  white-space: nowrap;
}

.tile-value {
  overflow: hidden;
  margin-top: 4px;
  color: var(--text);
  font-size: 20px;
  font-weight: 600;
  letter-spacing: -0.02em;
  line-height: 1.4;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.tile-divider {
  margin: 0 5px;
  color: var(--faint);
  font-size: 17px;
  font-weight: 400;
}

.tile-foot {
  margin-top: auto;
  padding-top: 4px;
  overflow: hidden;
  color: var(--muted);
  font-size: 12px;
  line-height: 1.4;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.tile-truncate {
  overflow: hidden;
  text-overflow: ellipsis;
}

.tile-split {
  display: flex;
  min-width: 0;
  align-items: center;
  gap: 6px;
  white-space: nowrap;
}

.tile-sep {
  color: var(--line-strong);
}

.tile-speed {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.tile-spark {
  width: 100%;
  height: 18px;
  margin-top: 2px;
  color: var(--text);
  opacity: 0.85;
}

.spark-muted {
  color: var(--muted);
}

@media (max-width: 900px) {
  .summary-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}

@media (max-width: 520px) {
  .summary-tile {
    min-height: 102px;
    padding: 12px;
  }

  .tile-value {
    font-size: 20px;
  }
}
</style>
