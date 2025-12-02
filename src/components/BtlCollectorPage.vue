<!-- Reminder: run `npm install chart.js vue-chartjs` to enable the charts. -->

<template>
  <div class="collector-page">
    <div class="page-content">
      <div class="top-bar">
        <span class="top-bar__title">MTD Services</span>
      </div>

      <div class="collector-card">
        <div class="card-header">
          <span class="card-header__title">MTD BTL Collector</span>
          <span class="info-icon">i</span>
        </div>
        <div class="card-body">
          <label class="form-label" for="serenity-board">Serenity board</label>
          <select
            id="serenity-board"
            v-model="selectedBoard"
            class="form-select"
          >
            <option disabled value="">Select</option>
            <option>Serenity S1.011</option>
            <option>Serenity S1.012</option>
            <option>Demo / Offline data</option>
          </select>
          <p v-if="showValidation" class="validation-message">
            Please select a Serenity board.
          </p>

          <div class="actions">
            <button type="button" class="btn secondary" @click="onReset">
              Reset
            </button>
            <button type="button" class="btn primary" @click="onDisplay">
              Display
            </button>
          </div>
        </div>
      </div>

      <div v-if="showMonitoring" class="monitoring-container">
        <div class="selected-board-banner">
          <div>
            <div class="banner-title">
              <span class="banner-label">Selected board:</span>
              <span class="banner-board">{{ selectedBoard }}</span>
            </div>
            <p class="banner-subtitle">
              Monitoring view is based on demo data (no live DAQ connection in this prototype).
            </p>
          </div>
          <span class="status-pill">Status: connected</span>
        </div>

        <div class="metric-card-row">
          <div class="metric-card">
            <p class="metric-label">AVG BTL TEMPERATURE</p>
            <p class="metric-value">23.4 °C</p>
            <p class="metric-caption">Updated 5 s ago</p>
          </div>
          <div class="metric-card">
            <p class="metric-label">MAX MODULE CURRENT</p>
            <p class="metric-value">1.8 A</p>
            <p class="metric-caption">RU 03 / CC 01</p>
          </div>
          <div class="metric-card">
            <p class="metric-label">RECENT TIMING HITS</p>
            <p class="metric-value">12.5 kHz</p>
            <p class="metric-caption">Demo hit rate (aggregated)</p>
          </div>
        </div>

        <div class="tabs">
          <button
            v-for="tab in tabs"
            :key="tab"
            type="button"
            class="tab"
            :class="{ active: activeTab === tab }"
            @click="activeTab = tab"
          >
            {{ tab }}
          </button>
        </div>

        <div class="tab-panel">
          <div v-if="activeTab === 'Overview'">
            <h3 class="panel-title">Overview</h3>
            <p class="panel-description">
              This overview shows a combined summary of recent monitoring values for the selected Serenity board.
            </p>
            <div class="overview-grid">
              <div class="overview-item">• Last run: #12345 (demo)</div>
              <div class="overview-item">• Data mode: monitoring demo</div>
              <div class="overview-item">• Active RUs: 4 / 4</div>
              <div class="overview-item">• Alerts in last hour: 0</div>
            </div>
          </div>

          <div v-else-if="activeTab === 'Temperatures'">
            <h3 class="panel-title">Temperatures</h3>
            <div class="chart-container">
              <Line
                :data="temperatureChartData"
                :options="temperatureChartOptions"
                class="monitoring-chart"
              />
            </div>
          </div>

          <div v-else-if="activeTab === 'Currents'">
            <h3 class="panel-title">Currents</h3>
            <div class="chart-container">
              <Bar
                :data="currentChartData"
                :options="currentChartOptions"
                class="monitoring-chart"
              />
            </div>
          </div>

          <div v-else-if="activeTab === 'Timing'">
            <h3 class="panel-title">Timing</h3>
            <div class="timing-heatmap">
              <div class="timing-heatmap-header">
                <div class="timing-heatmap-corner"></div>
                <div
                  v-for="bin in timingBins"
                  :key="bin"
                  class="timing-heatmap-header-cell"
                >
                  {{ bin }}
                </div>
              </div>
              <div
                v-for="(row, rowIndex) in timingMatrix"
                :key="timingChannels[rowIndex]"
                class="timing-heatmap-row"
              >
                <div class="timing-heatmap-channel">
                  {{ timingChannels[rowIndex] }}
                </div>
                <div
                  v-for="(value, colIndex) in row"
                  :key="colIndex"
                  class="timing-heatmap-cell"
                  :style="getTimingCellStyle(value)"
                >
                  {{ value }}
                </div>
              </div>
            </div>
          </div>

          <div v-else>
            <h3 class="panel-title">Logs</h3>
            <table class="logs-table">
              <thead>
                <tr>
                  <th>Timestamp</th>
                  <th>Level</th>
                  <th>Message</th>
                </tr>
              </thead>
              <tbody>
                <tr>
                  <td>2025-11-24 10:15:03</td>
                  <td>INFO</td>
                  <td>Demo monitoring session started for Serenity S1.011</td>
                </tr>
                <tr>
                  <td>2025-11-24 10:15:10</td>
                  <td>INFO</td>
                  <td>Fetched demo telemetry batch (64 samples)</td>
                </tr>
                <tr>
                  <td>2025-11-24 10:15:15</td>
                  <td>WARN</td>
                  <td>Temperature close to threshold on RU 03 (demo)</td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
// @ts-ignore chart stubs for prototype
import { Line, Bar } from 'vue-chartjs';
// @ts-ignore chart stubs for prototype
import {
  Chart,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  BarElement,
  Title,
  Tooltip,
  Legend,
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

const selectedBoard = ref('');
const showValidation = ref(false);
const showMonitoring = ref(false);
const activeTab = ref('Overview');
const tabs = ['Overview', 'Temperatures', 'Currents', 'Timing', 'Logs'];

const temperatureLabels = ['10:00', '10:05', '10:10', '10:15', '10:20', '10:25'];
const temperatureValues = [22.8, 23.1, 23.4, 23.7, 23.5, 23.4];
const currentLabels = ['RU 01', 'RU 02', 'RU 03', 'RU 04'];
const currentValues = [1.3, 1.5, 1.8, 1.4];
const timingBins = ['t0', 't1', 't2', 't3', 't4', 't5'];
const timingChannels = ['Ch 0-7', 'Ch 8-15', 'Ch 16-23', 'Ch 24-31'];
const timingMatrix = [
  [10, 20, 35, 50, 40, 25],
  [5, 15, 30, 45, 38, 20],
  [8, 18, 28, 42, 36, 22],
  [12, 25, 40, 55, 44, 30],
];

const temperatureChartData = {
  labels: temperatureLabels,
  datasets: [
    {
      label: 'BTL Temperature (°C)',
      data: temperatureValues,
      borderColor: '#0097a7',
      backgroundColor: 'rgba(0, 151, 167, 0.1)',
      tension: 0.3,
      pointRadius: 3,
    },
  ],
};

const temperatureChartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  scales: {
    x: {
      ticks: { color: '#607d8b' },
      grid: { display: false },
    },
    y: {
      ticks: { color: '#607d8b' },
      grid: { color: '#eceff1' },
    },
  },
  plugins: {
    legend: {
      labels: { color: '#455a64' },
    },
  },
};

const currentChartData = {
  labels: currentLabels,
  datasets: [
    {
      label: 'Module current (A)',
      data: currentValues,
      backgroundColor: 'rgba(0, 151, 167, 0.6)',
    },
  ],
};

const currentChartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  scales: {
    x: {
      ticks: { color: '#607d8b' },
      grid: { display: false },
    },
    y: {
      ticks: { color: '#607d8b' },
      grid: { color: '#eceff1' },
    },
  },
  plugins: {
    legend: {
      labels: { color: '#455a64' },
    },
  },
};

const getTimingCellStyle = (value: number) => {
  const max = 60;
  const ratio = Math.min(value / max, 1);
  const alpha = 0.15 + ratio * 0.7;
  return {
    backgroundColor: `rgba(0, 151, 167, ${alpha})`,
    color: ratio > 0.5 ? '#ffffff' : '#263238',
  };
};

const onReset = () => {
  selectedBoard.value = '';
  showValidation.value = false;
  showMonitoring.value = false;
  activeTab.value = 'Overview';
};

const onDisplay = () => {
  if (!selectedBoard.value) {
    showValidation.value = true;
    return;
  }

  showValidation.value = false;
  showMonitoring.value = true;
  activeTab.value = 'Overview';
  console.log('Display monitoring for board:', selectedBoard.value);
};
</script>

<style scoped>
.collector-page {
  background: #f5f9fc;
  min-height: 100vh;
  padding: 32px 24px 48px;
  box-sizing: border-box;
  color: #263238;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
    "Helvetica Neue", Arial, sans-serif;
  font-size: 14px;
}

.page-content {
  max-width: 1200px;
  margin: 0 auto;
}

.top-bar {
  height: 48px;
  background: #1c2833;
  border-radius: 4px;
  padding: 0 24px;
  display: flex;
  align-items: center;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.08);
}

.top-bar__title {
  color: #ffffff;
  font-weight: 600;
  font-size: 16px;
}

.collector-card {
  max-width: 900px;
  margin: 32px auto 0;
  background: #ffffff;
  border-radius: 6px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
  overflow: hidden;
}

.card-header {
  background: #cfeeff;
  height: 50px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 24px;
}

.card-header__title {
  font-weight: 600;
  color: #263238;
  font-size: 14px;
}

.info-icon {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #1976d2;
  color: #ffffff;
  font-size: 11px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
}

.card-body {
  padding: 24px;
}

.form-label {
  display: block;
  margin-bottom: 6px;
  font-weight: 500;
  color: #263238;
}

.form-select {
  width: 100%;
  height: 36px;
  border: 1px solid #cfd8dc;
  border-radius: 4px;
  padding: 0 12px;
  background: #ffffff;
  font-size: 14px;
  color: #263238;
  outline: none;
  transition: border 0.2s;
}

.form-select:focus {
  border-color: #90a4ae;
  box-shadow: 0 0 0 1px rgba(0, 151, 167, 0.15);
}

.validation-message {
  margin-top: 4px;
  font-size: 12px;
  color: #d32f2f;
}

.actions {
  margin-top: 16px;
  display: flex;
  justify-content: flex-end;
  gap: 8px;
}

.btn {
  border: 1px solid transparent;
  border-radius: 4px;
  height: 32px;
  min-width: 90px;
  padding: 0 16px;
  font-size: 14px;
  cursor: pointer;
  transition: background 0.2s, border-color 0.2s;
}

.btn.secondary {
  background: #ffffff;
  border-color: #b0bec5;
  color: #455a64;
  min-width: 80px;
}

.btn.secondary:hover {
  border-color: #90a4ae;
  background: #eceff1;
}

.btn.primary {
  background: #0097a7;
  border-color: #00838f;
  color: #ffffff;
  font-weight: 500;
}

.btn.primary:hover {
  background: #00838f;
}

.monitoring-container {
  max-width: 900px;
  margin: 24px auto 0;
  animation: fadeIn 0.25s ease;
}

.selected-board-banner {
  background: #e0f7fa;
  border: 1px solid #b2ebf2;
  border-radius: 4px;
  padding: 12px 16px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 16px;
}

.banner-title {
  font-weight: 600;
  color: #263238;
  margin-bottom: 4px;
}

.banner-label {
  margin-right: 6px;
}

.banner-board {
  color: #006064;
}

.banner-subtitle {
  margin: 0;
  font-size: 12px;
  color: #607d8b;
}

.status-pill {
  background: #c8e6c9;
  color: #2e7d32;
  padding: 4px 10px;
  border-radius: 999px;
  font-size: 12px;
  font-weight: 500;
}

.metric-card-row {
  display: flex;
  gap: 16px;
  margin-top: 16px;
  flex-wrap: wrap;
}

.metric-card {
  flex: 1 1 calc(33.333% - 10.7px);
  background: #ffffff;
  border: 1px solid #e0e0e0;
  border-radius: 4px;
  padding: 12px 14px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.04);
}

.metric-label {
  font-size: 11px;
  letter-spacing: 0.05em;
  color: #78909c;
  margin: 0 0 6px;
}

.metric-value {
  margin: 0;
  font-size: 20px;
  font-weight: 600;
  color: #263238;
}

.metric-caption {
  margin-top: 4px;
  font-size: 11px;
  color: #90a4ae;
}

.tabs {
  margin-top: 24px;
  border-bottom: 1px solid #cfd8dc;
  display: flex;
  gap: 16px;
  flex-wrap: wrap;
}

.tab {
  padding: 8px 12px;
  border-radius: 4px 4px 0 0;
  border: none;
  background: transparent;
  color: #607d8b;
  cursor: pointer;
  font-size: 13px;
  font-weight: 500;
}

.tab.active {
  background: #ffffff;
  color: #0097a7;
  border: 1px solid #cfd8dc;
  border-bottom: 2px solid #ffffff;
}

.tab-panel {
  background: #ffffff;
  border: 1px solid #cfd8dc;
  border-top: none;
  border-radius: 0 4px 4px 4px;
  padding: 16px 20px;
  min-height: 260px;
}

.panel-title {
  margin: 0 0 8px;
  font-size: 16px;
  font-weight: 600;
  color: #263238;
}

.panel-description {
  margin: 0 0 12px;
  color: #455a64;
}

.overview-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 8px 24px;
  margin-top: 8px;
}

.overview-item {
  color: #263238;
}

.chart-container {
  margin-top: 12px;
  height: 220px;
  border: 1px solid #e0e0e0;
  border-radius: 4px;
  background: #fafafa;
  padding: 8px;
}

.monitoring-chart {
  width: 100%;
  height: 100%;
}

.logs-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 13px;
}

.logs-table th,
.logs-table td {
  border-bottom: 1px solid #e0e0e0;
  padding: 10px 8px;
  text-align: left;
  color: #263238;
}

.logs-table tbody tr:hover {
  background: #f5f5f5;
}

@media (max-width: 768px) {
  .metric-card {
    flex: 1 1 100%;
  }

  .selected-board-banner {
    flex-direction: column;
    align-items: flex-start;
  }
}

.timing-heatmap {
  margin-top: 8px;
  border: 1px solid #cfd8dc;
  border-radius: 4px;
  overflow: hidden;
  font-size: 12px;
}

.timing-heatmap-header,
.timing-heatmap-row {
  display: grid;
  grid-template-columns: 120px repeat(6, 1fr);
}

.timing-heatmap-corner {
  border-right: 1px solid #eceff1;
}

.timing-heatmap-header {
  background: #e0f2f1;
  font-weight: 600;
  color: #006064;
}

.timing-heatmap-header-cell,
.timing-heatmap-channel,
.timing-heatmap-cell {
  padding: 6px 8px;
  border-bottom: 1px solid #eceff1;
  border-right: 1px solid #eceff1;
}

.timing-heatmap-channel {
  background: #f5f9fc;
  font-weight: 500;
  color: #455a64;
}

.timing-heatmap-row:last-child .timing-heatmap-cell,
.timing-heatmap-row:last-child .timing-heatmap-channel {
  border-bottom: none;
}

.timing-heatmap-header-cell:last-child,
.timing-heatmap-cell:last-child {
  border-right: none;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(6px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
</style>

