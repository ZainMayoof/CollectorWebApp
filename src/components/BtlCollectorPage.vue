<template>
  <div class="collector-page">
    <div class="page-header">
      <div>
        <h1 class="page-title">MTD BTL Collector Monitoring</h1>
        <p class="page-subtitle">Mock monitoring view with static demo data</p>
      </div>
      <div class="serenity-selector">
        <label for="serenity-board">Serenity board</label>
        <select id="serenity-board" v-model="selectedSerenity" class="form-select">
          <option disabled value="">Select a Serenity</option>
          <option v-for="option in serenityOptions" :key="option.id" :value="option.id">
            {{ option.label }}
          </option>
        </select>
      </div>
    </div>

    <div v-if="selectedSerenity" class="overview">
      <div class="overview-header">
        <div>
          <p class="overview-label">Selected Serenity</p>
          <p class="overview-title">{{ serenityMap[selectedSerenity].label }}</p>
          <p class="overview-updated">Last updated: {{ serenityMap[selectedSerenity].lastUpdated }}</p>
        </div>
        <span :class="['status-pill', serenityMap[selectedSerenity].status.toLowerCase()]">
          {{ serenityMap[selectedSerenity].status }}
        </span>
      </div>
      <div class="overview-grid">
        <div class="overview-card">
          <p class="card-label">CCs connected</p>
          <p class="card-value">{{ serenityMap[selectedSerenity].ccConnected }}</p>
        </div>
        <div class="overview-card">
          <p class="card-label">CCs in error</p>
          <p class="card-value">{{ serenityMap[selectedSerenity].ccError }}</p>
        </div>
        <div class="overview-card">
          <p class="card-label">Highest temperature</p>
          <p class="card-value">{{ serenityMap[selectedSerenity].highestTemperature }} °C</p>
        </div>
        <div class="overview-card">
          <p class="card-label">Total LV current</p>
          <p class="card-value">{{ serenityMap[selectedSerenity].totalCurrent }} A</p>
        </div>
      </div>
      <div class="cc-selector">
        <label for="cc">Channel Controller</label>
        <select id="cc" v-model="selectedCc" class="form-select">
          <option disabled value="">Select a CC</option>
          <option
            v-for="cc in serenityMap[selectedSerenity].ccNames"
            :key="cc"
            :value="cc"
          >
            {{ cc }}
          </option>
        </select>
      </div>
    </div>

    <div v-if="selectedCcData" class="cc-details">
      <div class="cc-details__header">
        <div>
          <p class="cc-label">Channel Controller</p>
          <h2 class="cc-title">{{ selectedCcData.ccName }}</h2>
          <p class="cc-subtitle">Serenity: {{ selectedCcData.serenity }}</p>
          <p class="cc-status" :class="selectedCcData.status.toLowerCase()">
            Status: {{ selectedCcData.status }}
          </p>
        </div>
        <div class="summary-box">
          <div class="summary-line">
            <span>Active alarms</span>
            <strong>{{ activeAlarmsCount }}</strong>
          </div>
          <div class="summary-line">
            <span>High hit-rate channels</span>
            <strong>{{ highHitChannels }}</strong>
          </div>
          <button class="btn" type="button" @click="exportSummary">Export summary</button>
        </div>
      </div>

      <div class="info-grid">
        <div class="info-card">
          <h3>CC Info</h3>
          <ul>
            <li><strong>Name:</strong> {{ selectedCcData.ccName }}</li>
            <li><strong>Serenity:</strong> {{ selectedCcData.serenity }}</li>
            <li><strong>Attached TOFHIR chips:</strong> {{ selectedCcData.tofhirChannels.length }}</li>
            <li><strong>Status:</strong> {{ selectedCcData.status }}</li>
          </ul>
        </div>
        <div class="info-card">
          <h3>Live Metrics</h3>
          <table class="data-table">
            <tbody>
              <tr v-for="metric in selectedCcData.liveMetrics" :key="metric.name">
                <td>{{ metric.name }}</td>
                <td class="value">{{ metric.value }} {{ metric.units }}</td>
              </tr>
            </tbody>
          </table>
        </div>
        <div class="info-card chart-card">
          <h3>Live Metric Trend</h3>
          <Line :data="liveChartData" :options="liveChartOptions" />
        </div>
      </div>

      <div class="charts-grid">
        <div class="chart-panel">
          <div class="chart-panel__header">
            <h3>Temperature history</h3>
            <span class="chip">10 samples</span>
          </div>
          <Line :data="temperatureHistoryData" :options="historyChartOptions" />
        </div>
        <div class="chart-panel">
          <div class="chart-panel__header">
            <h3>Voltage history</h3>
            <span class="chip">10 samples</span>
          </div>
          <Line :data="voltageHistoryData" :options="historyChartOptions" />
        </div>
      </div>

      <div class="tofhir">
        <div class="chart-panel">
          <div class="chart-panel__header">
            <h3>TOFHIR hits per channel</h3>
            <span class="chip">Channels 0-15</span>
          </div>
          <Bar :data="tofhirHitsData" :options="tofhirBarOptions" />
        </div>
        <div class="info-card">
          <h3>TOFHIR Reconstruction Overview</h3>
          <table class="data-table">
            <thead>
              <tr>
                <th>Channel</th>
                <th>Hits/s</th>
                <th>Avg time res (ps)</th>
                <th>Noise rate</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="channel in selectedCcData.tofhirChannels" :key="channel.channelId">
                <td>{{ channel.channelId }}</td>
                <td>{{ channel.hitsPerSecond }}</td>
                <td>{{ channel.avgTimeResolutionPs }}</td>
                <td>{{ channel.noiseRate }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div class="alarms-panel">
        <div class="alarms-header">
          <h3>Alarms</h3>
          <span class="chip">{{ selectedCcData.alarms.length }} items</span>
        </div>
        <div class="alarms-list">
          <div
            v-for="alarm in selectedCcData.alarms"
            :key="alarm.id"
            class="alarm-item"
            :class="alarm.severity.toLowerCase()"
          >
            <div class="alarm-top">
              <span class="alarm-severity">{{ alarm.severity }}</span>
              <span class="alarm-time">{{ alarm.timestamp }}</span>
            </div>
            <p class="alarm-source">{{ alarm.source }}</p>
            <p class="alarm-message">{{ alarm.message }}</p>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, ref, watch } from 'vue';
import { Bar, Line } from 'vue-chartjs';
import {
  BarElement,
  CategoryScale,
  Chart,
  Legend,
  LineElement,
  LinearScale,
  PointElement,
  Title,
  Tooltip,
} from 'chart.js';

Chart.register(
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  BarElement,
  Title,
  Tooltip,
  Legend,
);

type SerenityId = 'S1.2' | 'Z1.1' | 'Z1.2';

type StatusLevel = 'OK' | 'WARNING' | 'ERROR';

interface LiveMetric {
  name: string;
  value: number;
  units: string;
}

interface HistoricalPoint {
  timestamp: string;
  value: number;
}

interface TofhirChannel {
  channelId: number;
  hitsPerSecond: number;
  avgTimeResolutionPs: number;
  noiseRate: number;
}

interface Alarm {
  id: number;
  severity: 'INFO' | 'WARNING' | 'ALARM';
  source: string;
  message: string;
  timestamp: string;
}

interface CcMonitoring {
  serenity: SerenityId;
  ccName: string;
  status: StatusLevel;
  liveMetrics: LiveMetric[];
  historicalTemperature: HistoricalPoint[];
  historicalVoltage: HistoricalPoint[];
  tofhirChannels: TofhirChannel[];
  alarms: Alarm[];
}

interface SerenityOverview {
  label: string;
  status: StatusLevel;
  ccConnected: number;
  ccError: number;
  highestTemperature: number;
  totalCurrent: number;
  lastUpdated: string;
  ccNames: string[];
}

const serenityOptions: { id: SerenityId; label: string }[] = [
  { id: 'S1.2', label: 'Serenity S1.2' },
  { id: 'Z1.1', label: 'Serenity Z1.1' },
  { id: 'Z1.2', label: 'Serenity Z1.2' },
];

const serenityMap: Record<SerenityId, SerenityOverview> = {
  'S1.2': {
    label: 'Serenity S1.2',
    status: 'OK',
    ccConnected: 4,
    ccError: 0,
    highestTemperature: 37.2,
    totalCurrent: 3.8,
    lastUpdated: '2025-11-24 10:15:00',
    ccNames: ['CC1_T2', 'CC2_T2', 'CC3_T4', 'CC4_T4'],
  },
  'Z1.1': {
    label: 'Serenity Z1.1',
    status: 'WARNING',
    ccConnected: 3,
    ccError: 1,
    highestTemperature: 41.6,
    totalCurrent: 3.2,
    lastUpdated: '2025-11-24 10:10:00',
    ccNames: ['CC1_T1', 'CC2_T3', 'CC3_T1'],
  },
  'Z1.2': {
    label: 'Serenity Z1.2',
    status: 'OK',
    ccConnected: 5,
    ccError: 0,
    highestTemperature: 36.1,
    totalCurrent: 4.2,
    lastUpdated: '2025-11-24 10:12:00',
    ccNames: ['CC1_T2', 'CC2_T2', 'CC3_T2', 'CC4_T3', 'CC5_T5'],
  },
};

const ccMonitorings: CcMonitoring[] = [
  {
    serenity: 'S1.2',
    ccName: 'CC1_T2',
    status: 'OK',
    liveMetrics: [
      { name: 'Board temperature', value: 33.2, units: '°C' },
      { name: 'Front-end temp', value: 30.8, units: '°C' },
      { name: 'LV voltage', value: 2.5, units: 'V' },
      { name: 'HV bias', value: 120, units: 'V' },
      { name: 'LV current', value: 0.8, units: 'A' },
    ],
    historicalTemperature: Array.from({ length: 10 }, (_, idx) => ({
      timestamp: `10:${(idx + 1).toString().padStart(2, '0')}`,
      value: 31 + idx * 0.2,
    })),
    historicalVoltage: Array.from({ length: 10 }, (_, idx) => ({
      timestamp: `10:${(idx + 1).toString().padStart(2, '0')}`,
      value: 2.4 + idx * 0.01,
    })),
    tofhirChannels: Array.from({ length: 16 }, (_, channelId) => ({
      channelId,
      hitsPerSecond: 7500 + channelId * 150,
      avgTimeResolutionPs: 45 + (channelId % 5),
      noiseRate: 0.5 + channelId * 0.05,
    })),
    alarms: [
      {
        id: 1,
        severity: 'INFO',
        source: 'CC1_T2',
        message: 'Monitoring started for demo session',
        timestamp: '2025-11-24 10:12:01',
      },
      {
        id: 2,
        severity: 'WARNING',
        source: 'CC1_T2 / TOFHIR ch 3',
        message: 'Hit rate close to threshold',
        timestamp: '2025-11-24 10:14:30',
      },
    ],
  },
  {
    serenity: 'S1.2',
    ccName: 'CC2_T2',
    status: 'WARNING',
    liveMetrics: [
      { name: 'Board temperature', value: 37.1, units: '°C' },
      { name: 'Front-end temp', value: 35.4, units: '°C' },
      { name: 'LV voltage', value: 2.48, units: 'V' },
      { name: 'HV bias', value: 118, units: 'V' },
      { name: 'LV current', value: 0.95, units: 'A' },
    ],
    historicalTemperature: Array.from({ length: 10 }, (_, idx) => ({
      timestamp: `10:${(idx + 1).toString().padStart(2, '0')}`,
      value: 35 + idx * 0.25,
    })),
    historicalVoltage: Array.from({ length: 10 }, (_, idx) => ({
      timestamp: `10:${(idx + 1).toString().padStart(2, '0')}`,
      value: 2.45 + idx * 0.005,
    })),
    tofhirChannels: Array.from({ length: 16 }, (_, channelId) => ({
      channelId,
      hitsPerSecond: 6800 + channelId * 180,
      avgTimeResolutionPs: 50 + (channelId % 4),
      noiseRate: 0.7 + channelId * 0.04,
    })),
    alarms: [
      {
        id: 3,
        severity: 'WARNING',
        source: 'CC2_T2 / TOFHIR ch 7',
        message: 'Temperature above target window',
        timestamp: '2025-11-24 10:13:10',
      },
      {
        id: 4,
        severity: 'ALARM',
        source: 'CC2_T2',
        message: 'HV ripple detected in bias line',
        timestamp: '2025-11-24 10:14:55',
      },
    ],
  },
  {
    serenity: 'Z1.1',
    ccName: 'CC2_T3',
    status: 'ERROR',
    liveMetrics: [
      { name: 'Board temperature', value: 41.8, units: '°C' },
      { name: 'Front-end temp', value: 38.7, units: '°C' },
      { name: 'LV voltage', value: 2.38, units: 'V' },
      { name: 'HV bias', value: 112, units: 'V' },
      { name: 'LV current', value: 1.2, units: 'A' },
    ],
    historicalTemperature: Array.from({ length: 10 }, (_, idx) => ({
      timestamp: `10:${(idx + 1).toString().padStart(2, '0')}`,
      value: 37 + idx * 0.5,
    })),
    historicalVoltage: Array.from({ length: 10 }, (_, idx) => ({
      timestamp: `10:${(idx + 1).toString().padStart(2, '0')}`,
      value: 2.3 + idx * 0.02,
    })),
    tofhirChannels: Array.from({ length: 16 }, (_, channelId) => ({
      channelId,
      hitsPerSecond: 6200 + channelId * 120,
      avgTimeResolutionPs: 60 + (channelId % 6),
      noiseRate: 0.9 + channelId * 0.06,
    })),
    alarms: [
      {
        id: 5,
        severity: 'ALARM',
        source: 'CC2_T3 / TOFHIR ch 9',
        message: 'Noise rate too high',
        timestamp: '2025-11-24 10:09:50',
      },
      {
        id: 6,
        severity: 'ALARM',
        source: 'CC2_T3',
        message: 'Communication drop detected',
        timestamp: '2025-11-24 10:11:20',
      },
    ],
  },
  {
    serenity: 'Z1.2',
    ccName: 'CC4_T3',
    status: 'OK',
    liveMetrics: [
      { name: 'Board temperature', value: 32.4, units: '°C' },
      { name: 'Front-end temp', value: 30.6, units: '°C' },
      { name: 'LV voltage', value: 2.51, units: 'V' },
      { name: 'HV bias', value: 119, units: 'V' },
      { name: 'LV current', value: 0.76, units: 'A' },
    ],
    historicalTemperature: Array.from({ length: 10 }, (_, idx) => ({
      timestamp: `10:${(idx + 1).toString().padStart(2, '0')}`,
      value: 31 + idx * 0.15,
    })),
    historicalVoltage: Array.from({ length: 10 }, (_, idx) => ({
      timestamp: `10:${(idx + 1).toString().padStart(2, '0')}`,
      value: 2.48 + idx * 0.004,
    })),
    tofhirChannels: Array.from({ length: 16 }, (_, channelId) => ({
      channelId,
      hitsPerSecond: 7000 + channelId * 110,
      avgTimeResolutionPs: 48 + (channelId % 3),
      noiseRate: 0.55 + channelId * 0.03,
    })),
    alarms: [
      {
        id: 7,
        severity: 'INFO',
        source: 'CC4_T3',
        message: 'Routine calibration completed',
        timestamp: '2025-11-24 10:08:00',
      },
      {
        id: 8,
        severity: 'WARNING',
        source: 'CC4_T3 / TOFHIR ch 5',
        message: 'Hit rate approaching upper bound',
        timestamp: '2025-11-24 10:12:40',
      },
    ],
  },
];

const selectedSerenity = ref<SerenityId | ''>('');
const selectedCc = ref<string>('');

watch(selectedSerenity, (next) => {
  const available = next ? serenityMap[next].ccNames : [];
  selectedCc.value = available[0] ?? '';
});

const selectedCcData = computed(() =>
  ccMonitorings.find(
    (monitor) => monitor.serenity === selectedSerenity.value && monitor.ccName === selectedCc.value,
  ),
);

const liveChartData = computed(() => ({
  labels: selectedCcData.value?.liveMetrics.map((metric) => metric.name) ?? [],
  datasets: [
    {
      label: 'Live values',
      data: selectedCcData.value?.liveMetrics.map((metric) => metric.value) ?? [],
      backgroundColor: 'rgba(0, 151, 167, 0.2)',
      borderColor: '#0097a7',
      tension: 0.3,
    },
  ],
}));

const liveChartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: { legend: { display: false } },
  scales: {
    x: { ticks: { color: '#455a64', autoSkip: false }, grid: { display: false } },
    y: { ticks: { color: '#455a64' }, grid: { color: '#eceff1' } },
  },
};

const historyChartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: { legend: { display: false } },
  scales: {
    x: { ticks: { color: '#455a64' }, grid: { display: false } },
    y: { ticks: { color: '#455a64' }, grid: { color: '#eceff1' } },
  },
};

const temperatureHistoryData = computed(() => ({
  labels: selectedCcData.value?.historicalTemperature.map((p) => p.timestamp) ?? [],
  datasets: [
    {
      label: 'Temperature (°C)',
      data: selectedCcData.value?.historicalTemperature.map((p) => p.value) ?? [],
      borderColor: '#e65100',
      backgroundColor: 'rgba(230, 81, 0, 0.1)',
      tension: 0.25,
      pointRadius: 3,
    },
  ],
}));

const voltageHistoryData = computed(() => ({
  labels: selectedCcData.value?.historicalVoltage.map((p) => p.timestamp) ?? [],
  datasets: [
    {
      label: 'Voltage (V)',
      data: selectedCcData.value?.historicalVoltage.map((p) => p.value) ?? [],
      borderColor: '#1565c0',
      backgroundColor: 'rgba(21, 101, 192, 0.1)',
      tension: 0.25,
      pointRadius: 3,
    },
  ],
}));

const tofhirHitsData = computed(() => ({
  labels: selectedCcData.value?.tofhirChannels.map((ch) => ch.channelId.toString()) ?? [],
  datasets: [
    {
      label: 'Hits per second',
      data: selectedCcData.value?.tofhirChannels.map((ch) => ch.hitsPerSecond) ?? [],
      backgroundColor: 'rgba(0, 150, 136, 0.6)',
    },
  ],
}));

const tofhirBarOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: { legend: { display: false } },
  scales: {
    x: { ticks: { color: '#455a64' } },
    y: { ticks: { color: '#455a64' }, grid: { color: '#eceff1' } },
  },
};

const activeAlarmsCount = computed(
  () => selectedCcData.value?.alarms.filter((alarm) => alarm.severity !== 'INFO').length ?? 0,
);

const highHitChannels = computed(
  () => selectedCcData.value?.tofhirChannels.filter((channel) => channel.hitsPerSecond > 8000).length ?? 0,
);

const exportSummary = () => {
  if (!selectedCcData.value) return;

  const latestTemperature = selectedCcData.value.historicalTemperature.slice(-1)[0];
  const latestVoltage = selectedCcData.value.historicalVoltage.slice(-1)[0];

  const payload = {
    serenity: selectedCcData.value.serenity,
    cc: selectedCcData.value.ccName,
    status: selectedCcData.value.status,
    liveMetrics: selectedCcData.value.liveMetrics,
    latestTemperature,
    latestVoltage,
    highHitChannels: selectedCcData.value.tofhirChannels.filter((channel) => channel.hitsPerSecond > 8000),
    alarms: selectedCcData.value.alarms,
  };

  const blob = new Blob([JSON.stringify(payload, null, 2)], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const link = document.createElement('a');
  link.href = url;
  link.download = `summary_${payload.serenity}_${payload.cc}.json`;
  link.click();
  URL.revokeObjectURL(url);
};
</script>

<style scoped>
.collector-page {
  padding: 32px 24px 48px;
  background: #f4f7fb;
  min-height: 100vh;
  color: #263238;
  font-family: 'Inter', 'Segoe UI', Roboto, Arial, sans-serif;
}

.page-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-end;
  gap: 16px;
  flex-wrap: wrap;
}

.page-title {
  margin: 0;
  font-size: 24px;
  font-weight: 700;
  color: #1c2833;
}

.page-subtitle {
  margin: 4px 0 0;
  color: #607d8b;
}

.serenity-selector {
  display: flex;
  flex-direction: column;
  gap: 6px;
  min-width: 240px;
}

.form-select {
  border: 1px solid #cfd8dc;
  border-radius: 6px;
  height: 38px;
  padding: 0 12px;
  background: #ffffff;
  font-size: 14px;
  color: #263238;
}

.overview {
  margin-top: 24px;
  background: #ffffff;
  border: 1px solid #e0e6ed;
  border-radius: 10px;
  padding: 16px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.04);
}

.overview-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  flex-wrap: wrap;
}

.overview-label {
  margin: 0;
  font-size: 12px;
  color: #607d8b;
}

.overview-title {
  margin: 2px 0;
  font-size: 18px;
  font-weight: 700;
}

.overview-updated {
  margin: 0;
  color: #78909c;
  font-size: 12px;
}

.status-pill {
  padding: 6px 12px;
  border-radius: 999px;
  font-weight: 700;
  font-size: 12px;
  text-transform: uppercase;
}

.status-pill.ok {
  background: #e0f2f1;
  color: #00695c;
}

.status-pill.warning {
  background: #fff3e0;
  color: #e65100;
}

.status-pill.error {
  background: #ffebee;
  color: #c62828;
}

.overview-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
  gap: 12px;
  margin-top: 12px;
}

.overview-card {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 12px;
}

.card-label {
  margin: 0;
  font-size: 12px;
  color: #607d8b;
}

.card-value {
  margin: 6px 0 0;
  font-size: 20px;
  font-weight: 700;
}

.cc-selector {
  margin-top: 16px;
  display: flex;
  flex-direction: column;
  gap: 6px;
  max-width: 260px;
}

.cc-details {
  margin-top: 20px;
  background: #ffffff;
  border: 1px solid #e0e6ed;
  border-radius: 12px;
  padding: 16px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.04);
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.cc-details__header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 16px;
  flex-wrap: wrap;
}

.cc-label {
  margin: 0;
  color: #607d8b;
  font-size: 12px;
}

.cc-title {
  margin: 4px 0 6px;
  font-size: 22px;
  font-weight: 800;
}

.cc-subtitle {
  margin: 0;
  color: #78909c;
}

.cc-status {
  margin: 8px 0 0;
  padding: 4px 10px;
  border-radius: 6px;
  display: inline-block;
  font-weight: 700;
}

.cc-status.ok {
  background: #e0f2f1;
  color: #00796b;
}

.cc-status.warning {
  background: #fff3e0;
  color: #ef6c00;
}

.cc-status.error {
  background: #ffebee;
  color: #c62828;
}

.summary-box {
  background: #0c253f;
  color: #ffffff;
  border-radius: 10px;
  padding: 12px 14px;
  min-width: 220px;
  box-shadow: 0 8px 16px rgba(12, 37, 63, 0.2);
}

.summary-line {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
  font-size: 14px;
}

.summary-line strong {
  font-size: 16px;
}

.summary-box .btn {
  width: 100%;
  background: #29b6f6;
  color: #0c253f;
  border: none;
  border-radius: 8px;
  padding: 8px 10px;
  font-weight: 700;
  cursor: pointer;
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
  gap: 12px;
}

.info-card {
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 12px;
  background: #fafbfc;
}

.info-card h3 {
  margin: 0 0 10px;
  font-size: 16px;
}

.info-card ul {
  padding-left: 16px;
  margin: 0;
  color: #455a64;
}

.data-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 13px;
}

.data-table th,
.data-table td {
  padding: 8px 6px;
  border-bottom: 1px solid #e0e6ed;
  text-align: left;
}

.data-table th {
  color: #607d8b;
  font-weight: 600;
}

.data-table .value {
  text-align: right;
  font-weight: 700;
}

.chart-card {
  min-height: 240px;
}

.chart-card canvas {
  height: 220px !important;
}

.charts-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 12px;
}

.chart-panel {
  background: #ffffff;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 12px;
  min-height: 260px;
}

.chart-panel__header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 8px;
}

.chart-panel h3 {
  margin: 0;
  font-size: 16px;
}

.chip {
  background: #e3f2fd;
  color: #1565c0;
  padding: 4px 8px;
  border-radius: 999px;
  font-size: 12px;
}

.chart-panel canvas {
  height: 220px !important;
}

.tofhir {
  display: grid;
  grid-template-columns: 1.1fr 1fr;
  gap: 12px;
}

.alarms-panel {
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  background: #ffffff;
  padding: 12px;
}

.alarms-header {
  display: flex;
  align-items: center;
  gap: 8px;
}

.alarms-header h3 {
  margin: 0;
}

.alarms-list {
  margin-top: 10px;
  max-height: 220px;
  overflow-y: auto;
  display: grid;
  gap: 8px;
}

.alarm-item {
  border-radius: 8px;
  padding: 10px;
  border: 1px solid #e2e8f0;
}

.alarm-item.info {
  background: #e3f2fd;
  border-color: #bbdefb;
}

.alarm-item.warning {
  background: #fff3e0;
  border-color: #ffe0b2;
}

.alarm-item.alarm {
  background: #ffebee;
  border-color: #ffcdd2;
}

.alarm-top {
  display: flex;
  justify-content: space-between;
  font-size: 12px;
  color: #455a64;
}

.alarm-source {
  margin: 6px 0 0;
  font-weight: 700;
}

.alarm-message {
  margin: 4px 0 0;
  color: #263238;
}

.btn {
  cursor: pointer;
  transition: transform 0.15s ease;
}

.btn:hover {
  transform: translateY(-1px);
}

@media (max-width: 900px) {
  .tofhir {
    grid-template-columns: 1fr;
  }
}
</style>
