<template>
  <div class="data-management">
    <div class="header-section card">
      <div class="header-content">
        <h3 class="page-title">{{ title || '环境数据监控' }}</h3>
        <p class="subtitle">{{ subtitle || '监控和管理环境传感器数据' }}</p>
      </div>
    </div>

    <!-- 主卡片容器 -->
    <div class="main-card-container card">
      <!-- 切换按钮 -->
      <div class="main-card-tabs">
        <button 
          class="main-tab-btn" 
          :class="{ 'active': activeMainCard === 'chart' }" 
          @click="switchMainCard('chart')"
        >
          环境数据报表
        </button>
        <button 
          class="main-tab-btn" 
          :class="{ 'active': activeMainCard === 'sensor' }" 
          @click="switchMainCard('sensor')"
        >
          传感器记录
        </button>
        <button 
          class="main-tab-btn" 
          :class="{ 'active': activeMainCard === 'alarm' }" 
          @click="switchMainCard('alarm')"
        >
          预警记录
        </button>
      </div>

      <!-- 环境数据曲线卡片 -->
      <div class="card-content" v-show="activeMainCard === 'chart'">
        <div class="data-charts">
          <div class="chart-section">
            <h3>环境数据曲线</h3>
            <div class="chart-container" v-if="showChart">
              <Line :data="chartData" :options="chartOptions" />
            </div>
          </div>
          <div class="chart-section">
            <h3>风险报表</h3>
            <div class="report-content" v-if="reportGenerated">
              <h4>{{ reportTitle }}</h4>
              <div class="report-stats">
                <div class="stat-card">
                  <h5>高频风险点</h5>
                  <ul>
                    <li v-for="(risk, index) in highRiskPoints" :key="index">
                      <span class="risk-device">{{ risk.device }}</span> 
                      <span class="risk-location">{{ risk.location }}</span>
                      <span class="risk-type">{{ risk.risk }}</span>
                      <span class="risk-level" :class="risk.level.toLowerCase()">{{ risk.level }}</span>
                      <span class="risk-time">{{ risk.time }}</span>
                    </li>
                  </ul>
                </div>
                
                <div class="stat-card">
                  <h5>风险趋势分析</h5>
                  <p>{{ riskTrendAnalysis }}</p>
                </div>
                
                <div class="stat-card">
                  <h5>管理建议</h5>
                  <p>{{ managementSuggestions }}</p>
                </div>
              </div>
            </div>
            <div v-else class="generate-report-section">
              <div class="report-type-selector">
                <label>报表类型：</label>
                <select v-model="reportType">
                  <option value="daily">日报</option>
                  <option value="weekly">周报</option>
                  <option value="monthly">月报</option>
                </select>
              </div>
              <button @click="generateReport" class="btn btn-primary">生成报表</button>
            </div>
          </div>
        </div>
      </div>

      <!-- 传感器记录卡片 -->
      <div class="card-content" v-show="activeMainCard === 'sensor'">
        <div class="data-sequences">
          <table class="table">
            <thead>
              <tr>
                <th>设备编号</th>
                <th>位置</th>
                <th>温度 (°C)</th>
                <th>湿度 (%)</th>
                <th>易燃气体</th>
                <th>烟雾</th>
                <th>更新时间</th>
                <th>操作</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="sensor in sensors" :key="sensor.id">
                <td>{{ sensor.deviceId }}</td>
                <td>{{ sensor.location }}</td>
                <td :class="sensor.temperature > 60 ? 'high-value' : ''">{{ sensor.temperature }}°C</td>
                <td :class="sensor.humidity > 80 || sensor.humidity < 20 ? 'high-value' : ''">{{ sensor.humidity }}%</td>
                <td :class="sensor.gas > 50 ? 'high-value' : ''">{{ sensor.gas }}ppm</td>
                <td :class="sensor.smoke ? 'high-value' : ''">{{ sensor.smoke ? '是' : '否' }}</td>
                <td>{{ sensor.updateTime }}</td>
                <td>
                  <button class="btn btn-small btn-primary" @click="viewSensor(sensor)">查看</button>
                  <button class="btn btn-small btn-secondary" @click="editSensor(sensor)">编辑</button>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- 预警记录卡片 -->
      <div class="card-content" v-show="activeMainCard === 'alarm'">
        <div class="data-sequences">
          <table class="table">
            <thead>
              <tr>
                <th>设备名</th>
                <th>预警项</th>
                <th>预警级别</th>
                <th>预警值</th>
                <th>发生时间</th>
                <th>处理状态</th>
                <th>操作</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="alarm in alarms" :key="alarm.id">
                <td>{{ alarm.deviceName }}</td>
                <td>{{ alarm.alarmItem }}</td>
                <td>
                  <span :class="['alarm-level', alarm.level]">{{ alarm.levelText }}</span>
                </td>
                <td>{{ alarm.value }}</td>
                <td>{{ alarm.time }}</td>
                <td>
                  <span :class="['status-badge', alarm.status]">{{ alarm.statusText }}</span>
                </td>
                <td>
                  <button class="btn btn-small btn-primary" @click="viewAlarm(alarm)">详情</button>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>

    <!-- 报告导出 -->
    <div class="export-section card" v-if="reportGenerated">
      <div class="export-header">
        <h3>报告导出</h3>
        <div class="export-controls">
          <button @click="exportPDF" class="btn btn-secondary pdf">
            <i class="icon">📄</i> 导出 PDF
          </button>
          <button @click="exportWord" class="btn btn-primary word">
            <i class="icon">📝</i> 导出 Word
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, onUnmounted, watch } from 'vue';
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
  Filler,
} from 'chart.js';
import { Line } from 'vue-chartjs';

ChartJS.register(
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
  Filler
);

// 定义 props
const props = defineProps({
  title: {
    type: String,
    default: '环境数据监控'
  },
  subtitle: {
    type: String,
    default: '监控和管理环境传感器数据'
  },
  apiEndpoints: {
    type: Array,
    default: () => [
      '/api/environment-data',
      '/api/robots',
      '/api/battery-summary'
    ]
  },
  initialCategory: {
    type: String,
    default: ''
  },
  autoRefresh: {
    type: Boolean,
    default: false
  },
  refreshInterval: {
    type: Number,
    default: 30000
  }
});

// 定义 emits
const emit = defineEmits(['data-loaded', 'category-selected', 'error']);

// 响应式数据
const startDate = ref('');
const endDate = ref('');
const selectedRegion = ref('');
const reportType = ref('daily');
const showChart = ref(true);
const reportGenerated = ref(false);
const showCategoryDropdown = ref(true);
const selectedCategory = ref(props.initialCategory);

// 自动刷新定时器
let refreshTimer = null;

// 分类列表数据
const categoriesList = ref([
  { id: 1, name: '温度数据' },
  { id: 2, name: '湿度数据' },
  { id: 3, name: '气体浓度' },
  { id: 4, name: '烟雾检测' },
  { id: 5, name: '预警记录' },
  { id: 6, name: '设备状态' }
]);

// 主卡片切换：'chart' 图表 / 'sensor' 传感器 / 'alarm' 预警
const activeMainCard = ref('chart');

// 切换主卡片
const switchMainCard = (type) => {
  activeMainCard.value = type;
};

// 区域数据
const regions = ref([
  { id: 'region1', name: 'A 区' },
  { id: 'region2', name: 'B 区' },
  { id: 'region3', name: 'C 区' },
  { id: 'region4', name: 'D 区' },
]);

// 图表数据
const chartData = ref({
  labels: ['1 月', '2 月', '3 月', '4 月', '5 月', '6 月'],
  datasets: [
    {
      label: '环境数据',
      backgroundColor: 'rgba(54, 162, 235, 0.2)',
      borderColor: 'rgba(54, 162, 235, 1)',
      data: [],
      fill: true,
      tension: 0.4,
    },
    {
      label: '温度',
      backgroundColor: 'rgba(255, 99, 132, 0.2)',
      borderColor: 'rgba(255, 99, 132, 1)',
      data: [],
      fill: true,
      tension: 0.4,
    },
    {
      label: '湿度',
      backgroundColor: 'rgba(75, 192, 192, 0.2)',
      borderColor: 'rgba(75, 192, 192, 1)',
      data: [],
      fill: true,
      tension: 0.4,
    }
  ],
});

const chartOptions = ref({
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: {
      position: 'top',
    },
    title: {
      display: true,
      text: '环境数据趋势图'
    }
  },
  scales: {
    y: {
      beginAtZero: true
    }
  }
});

// 报表数据
const reportTitle = ref('');
const highRiskPoints = ref([
  { device: '设备 A01', location: '一楼', risk: '高温异常', level: '高', time: '2026-01-15 14:30' },
  { device: '设备 B05', location: '二楼', risk: '振动超标', level: '中', time: '2026-01-15 10:15' },
  { device: '设备 C12', location: '三楼', risk: '压力异常', level: '高', time: '2026-01-15 09:45' },
  { device: '设备 D03', location: '四楼', risk: '温湿度异常', level: '低', time: '2026-01-14 16:20' },
  { device: '设备 E07', location: '五楼', risk: '空气质量差', level: '中', time: '2026-01-14 11:10' }
]);
const riskTrendAnalysis = ref('根据近期数据分析，设备故障率较上月下降 15%，但仍需重点关注一楼的高温问题。高温风险点数量较上周增加了 20%，建议加强相关设备的日常巡检。');
const managementSuggestions = ref('1. 加强一楼设备的日常巡检频次\n2. 对高温异常设备进行专项维保\n3. 优化设备运行参数，减少非计划停机时间\n4. 增加温度监控点位，提高监测精度\n5. 完善应急预案，提高应急响应速度');

// 传感器记录数据
const sensors = ref([
  { id: 1, deviceId: 'T001', location: '一楼大厅', temperature: 25, humidity: 45, gas: 15, smoke: false, updateTime: '2026-01-28 14:20' },
  { id: 2, deviceId: 'T002', location: '二楼办公室', temperature: 23, humidity: 50, gas: 10, smoke: false, updateTime: '2026-01-28 14:15' },
  { id: 3, deviceId: 'T003', location: '三楼实验室', temperature: 65, humidity: 85, gas: 60, smoke: true, updateTime: '2026-01-28 14:25' },
  { id: 4, deviceId: 'T004', location: '地下室', temperature: 18, humidity: 90, gas: 5, smoke: false, updateTime: '2026-01-28 14:10' },
  { id: 5, deviceId: 'T005', location: '天台', temperature: 20, humidity: 40, gas: 20, smoke: false, updateTime: '2026-01-28 14:18' }
]);

// 预警记录数据
const alarms = ref([
  { id: 1, deviceName: '设备 A01', alarmItem: '温度', level: 'high', levelText: '高温', value: '65°C', time: '2026-01-28 10:30', status: 'pending', statusText: '待处理' },
  { id: 2, deviceName: '设备 B05', alarmItem: '湿度', level: 'high', levelText: '高湿', value: '85%', time: '2026-01-28 11:15', status: 'handled', statusText: '已处理' },
  { id: 3, deviceName: '设备 C12', alarmItem: '电量', level: 'low', levelText: '低电', value: '15%', time: '2026-01-28 09:45', status: 'pending', statusText: '待处理' },
  { id: 4, deviceName: '设备 D03', alarmItem: '可燃物', level: 'high', levelText: '危险', value: '70ppm', time: '2026-01-28 12:20', status: 'handling', statusText: '处理中' },
  { id: 5, deviceName: '设备 E07', alarmItem: '温度', level: 'critical', levelText: '危急', value: '85°C', time: '2026-01-27 16:30', status: 'handled', statusText: '已处理' }
]);

// 格式化日期
const formatDate = (date) => {
  const year = date.getFullYear();
  const month = String(date.getMonth() + 1).padStart(2, '0');
  const day = String(date.getDate()).padStart(2, '0');
  return `${year}-${month}-${day}`;
};

// 监听 selectedCategory 变化
watch(selectedCategory, (newVal) => {
  emit('category-selected', newVal);
});

// 分类选择相关方法
const toggleCategoryDropdown = () => {
  showCategoryDropdown.value = !showCategoryDropdown.value;
};

// 点击外部关闭菜单
const handleClickOutside = (event) => {
  const dropdown = document.querySelector('.category-dropdown');
  if (dropdown && !dropdown.contains(event.target)) {
    showCategoryDropdown.value = false;
  }
};

// 键盘事件处理
const handleKeydown = (event) => {
  if (event.key === 'Escape') {
    showCategoryDropdown.value = false;
  }
};

// 合并后的 onMounted
onMounted(() => {
  console.log('数据管理页面已加载');
  
  const today = new Date();
  const oneWeekAgo = new Date(today.getTime() - 7 * 24 * 60 * 60 * 1000);
  
  startDate.value = formatDate(oneWeekAgo);
  endDate.value = formatDate(today);
  
  document.addEventListener('click', handleClickOutside);
  document.addEventListener('keydown', handleKeydown);
  
  fetchEnvironmentData();
  
  if (props.autoRefresh) {
    refreshTimer = setInterval(fetchEnvironmentData, props.refreshInterval);
  }
});

// 组件卸载前移除事件监听
onUnmounted(() => {
  document.removeEventListener('click', handleClickOutside);
  document.removeEventListener('keydown', handleKeydown);
  if (refreshTimer) {
    clearInterval(refreshTimer);
  }
});

const selectCategory = (categoryName) => {
  selectedCategory.value = categoryName;
  showCategoryDropdown.value = false;
  emit('category-selected', categoryName);
};

// 生成报表
const generateReport = () => {
  reportTitle.value = `${getReportTypeName()} - ${new Date().toLocaleDateString('zh-CN')}`;
  reportGenerated.value = true;
};

const getReportTypeName = () => {
  switch(reportType.value) {
    case 'daily': return '每日风险报告';
    case 'weekly': return '每周风险报告';
    case 'monthly': return '每月风险报告';
    default: return '风险报告';
  }
};

// 导出功能
const exportPDF = () => {
  alert('正在导出 PDF 报告...');
};

const exportWord = () => {
  alert('正在导出 Word 报告...');
};

// 其他操作方法
const importData = () => {
  alert('导入数据功能');
};

const exportData = () => {
  alert('导出数据功能');
};

const createCategory = () => {
  alert('新建分类功能');
};

const viewSensor = (sensor) => {
  alert(`查看传感器：${sensor.deviceId} - ${sensor.location}`);
};

const editSensor = (sensor) => {
  alert(`编辑传感器：${sensor.deviceId} - ${sensor.location}`);
};

const viewAlarm = (alarm) => {
  alert(`预警详情：${alarm.deviceName} - ${alarm.alarmItem}`);
};

const handleAlarm = (alarm) => {
  alert(`处理预警：${alarm.deviceName} - ${alarm.alarmItem}`);
  alarm.status = 'handling';
  alarm.statusText = '处理中';
};

// 获取环境数据
const fetchEnvironmentData = async () => {
  try {
    let response;
    let apiUrl;
    
    for (const endpoint of props.apiEndpoints) {
      try {
        apiUrl = endpoint;
        response = await fetch(apiUrl, {
          method: 'GET',
          headers: {
            'Content-Type': 'application/json'
          }
        });
        
        if (response.ok) {
          break;
        }
      } catch (error) {
        console.warn(`无法连接到 ${apiUrl}:`, error);
        continue;
      }
    }
    
    if (!response || !response.ok) {
      throw new Error('无法从任何 API 端点获取数据');
    }

    const data = await response.json();
    console.log('环境数据:', data);
    
    updateChartData(data);
    emit('data-loaded', data);
  } catch (error) {
    console.error('获取环境数据失败:', error);
    alert('获取环境数据失败：' + error.message);
    emit('error', error);
  }
};

// 更新图表数据
const updateChartData = (data) => {
  if (!data) {
    console.warn('环境数据为空');
    return;
  }
  
  if (data.labels && Array.isArray(data.datasets)) {
    chartData.value.labels = data.labels || chartData.value.labels;
    
    for (let i = 0; i < chartData.value.datasets.length; i++) {
      if (data.datasets[i]) {
        Object.assign(chartData.value.datasets[i], data.datasets[i], {
          fill: true,
          tension: 0.4
        });
      }
    }
    
    if (data.datasets.length > chartData.value.datasets.length) {
      for (let i = chartData.value.datasets.length; i < data.datasets.length; i++) {
        chartData.value.datasets.push({
          ...data.datasets[i],
          fill: true,
          tension: 0.4
        });
      }
    }
  } else if (Array.isArray(data) && data.length > 0) {
    if (data[0].hasOwnProperty('temperature') || data[0].hasOwnProperty('humidity') || data[0].hasOwnProperty('gas')) {
      const labels = data.map((_, index) => `传感器${index + 1}`);
      const temperatureData = data.map(item => item.temperature || 0);
      const humidityData = data.map(item => item.humidity || 0);
      const gasData = data.map(item => item.gas || 0);
      
      chartData.value.labels = labels;
      chartData.value.datasets[0].data = temperatureData;
      chartData.value.datasets[1].data = humidityData;
      chartData.value.datasets[2].data = gasData;
    }
  }
  
  console.log('图表数据已更新', chartData.value);
};
</script>

<style scoped>
.data-management {
  padding: 0 20px;
  max-width: 1200px;
  margin: 0 auto;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
}

.card {
  background: white;
  border-radius: 12px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  margin-bottom: 20px;
  overflow: visible;
}

.header-section {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px;
  margin-bottom: 20px;
  overflow: visible;
  position: relative;
}

.header-content .page-title {
  color: #2c3e50;
  margin-top: 0;
  margin-bottom: 5px;
  font-size: 1.8rem;
  font-weight: 600;
}

.header-content .subtitle {
  color: #7f8c8d;
  margin-top: 0;
  font-size: 1rem;
}

/* 主卡片容器 */
.main-card-container {
  padding: 0;
  overflow: hidden;
}

/* 主卡片切换按钮 */
.main-card-tabs {
  display: flex;
  gap: 5px;
  padding: 20px 20px 0 20px;
  border-bottom: 1px solid #eee;
  background: #fafbfc;
}

.main-tab-btn {
  padding: 12px 28px;
  border: none;
  background: #e9ecef;
  color: #7f8c8d;
  font-size: 15px;
  font-weight: 500;
  cursor: pointer;
  border-radius: 8px 8px 0 0;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  gap: 8px;
}

.main-tab-btn:hover {
  color: #2c3e50;
  background: #dee2e6;
}

.main-tab-btn.active {
  color: #42b983;
  background: white;
  border-bottom: 2px solid #42b983;
  margin-bottom: -1px;
}

/* 卡片内容区域 */
.card-content {
  padding: 20px;
  animation: fadeIn 0.3s ease;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* 图表区域 */
.data-charts {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;
  margin-bottom: 0;
  padding: 0;
}

.chart-section h3 {
  margin-bottom: 15px;
  color: #2c3e50;
  font-size: 1.2rem;
  font-weight: 600;
}

.chart-container {
  height: 300px;
  margin-top: 10px;
}

/* 表格区域 */
.data-sequences {
  padding: 0;
}

.table {
  width: 100%;
  border-collapse: collapse;
  margin-bottom: 0;
}

.table th, .table td {
  padding: 12px 15px;
  text-align: left;
  border-bottom: 1px solid #eee;
}

.table th {
  background-color: #f8f9fa;
  font-weight: 600;
  color: #2c3e50;
}

.high-value {
  color: #e74c3c;
  font-weight: bold;
}

/* 按钮样式 */
.btn {
  padding: 10px 20px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 500;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  gap: 8px;
}

.btn-small {
  padding: 6px 12px;
  font-size: 0.85rem;
  margin-right: 5px;
}

.btn-primary {
  background: linear-gradient(135deg, #42b983 0%, #2d7c5f 100%);
  color: white;
}

.btn-secondary {
  background: linear-gradient(135deg, #95a5a6 0%, #7f8c8d 100%);
  color: white;
}

/* 报表样式 */
.report-content h4 {
  color: #333;
  margin-top: 0;
  margin-bottom: 15px;
  font-size: 1.1rem;
}

.stat-card {
  background: #f8f9fa;
  border: 1px solid #eee;
  border-radius: 8px;
  padding: 12px;
  margin-bottom: 12px;
}

.stat-card h5 {
  margin-top: 0;
  margin-bottom: 8px;
  color: #42b983;
  font-size: 1rem;
  font-weight: 600;
}

.stat-card ul {
  padding-left: 0;
  margin: 8px 0;
}

.stat-card li {
  list-style: none;
  padding: 8px;
  border-bottom: 1px solid #eee;
  display: flex;
  justify-content: space-between;
  flex-wrap: wrap;
}

.risk-level {
  font-weight: bold;
  padding: 2px 8px;
  border-radius: 12px;
  font-size: 0.8rem;
}

.risk-level.high {
  background: rgba(231, 76, 60, 0.2);
  color: #e74c3c;
}

.risk-level.medium {
  background: rgba(243, 156, 18, 0.2);
  color: #f39c12;
}

.risk-level.low {
  background: rgba(52, 152, 219, 0.2);
  color: #3498db;
}

.generate-report-section {
  padding: 20px;
  text-align: center;
  background: #f8f9fa;
  border-radius: 8px;
}

.report-type-selector {
  margin-bottom: 15px;
}

.report-type-selector label {
  margin-right: 10px;
  font-weight: 500;
  color: #2c3e50;
}

.report-type-selector select {
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 14px;
}

/* 预警级别样式 */
.alarm-level {
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 0.8rem;
  font-weight: bold;
}

.alarm-level.high {
  background: rgba(243, 156, 18, 0.2);
  color: #f39c12;
}

.alarm-level.critical {
  background: rgba(231, 76, 60, 0.2);
  color: #e74c3c;
}

.alarm-level.low {
  background: rgba(52, 152, 219, 0.2);
  color: #3498db;
}

/* 状态徽章 */
.status-badge {
  padding: 4px 10px;
  border-radius: 20px;
  font-size: 0.8rem;
  font-weight: 500;
  display: inline-block;
}

.status-badge.pending {
  background: rgba(243, 156, 18, 0.2);
  color: #f39c12;
}

.status-badge.handling {
  background: rgba(52, 152, 219, 0.2);
  color: #3498db;
}

.status-badge.handled {
  background: rgba(46, 204, 113, 0.2);
  color: #27ae60;
}

/* 导出区域 */
.export-section {
  padding: 20px;
}

.export-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.export-header h3 {
  color: #2c3e50;
  margin: 0;
  font-size: 1.2rem;
  font-weight: 600;
}

.export-controls {
  display: flex;
  gap: 10px;
}

/* 响应式 */
@media (max-width: 992px) {
  .header-section {
    flex-direction: column;
    align-items: flex-start;
    gap: 20px;
  }
  
  .data-charts {
    grid-template-columns: 1fr;
  }
  
  .table {
    display: block;
    overflow-x: auto;
    white-space: nowrap;
  }
}

@media (max-width: 768px) {
  .main-card-tabs {
    flex-direction: column;
    gap: 5px;
  }
  
  .main-tab-btn {
    width: 100%;
    justify-content: center;
    border-radius: 8px;
    margin-bottom: 0;
  }
  
  .main-tab-btn.active {
    border-bottom: none;
    border-left: 3px solid #42b983;
  }
  
  .export-controls {
    flex-direction: column;
    width: 100%;
  }
  
  .export-controls > button {
    width: 100%;
  }
}
</style>