<template>
    <!-- 其他内容区域 -->
    <div class="content-grid">
      <!-- 左侧：数字孪生地图和机器人位置 -->
      <div class="map-section card">
        <div class="map-header">
          <div class="map-title-group">
            <h3>数字孪生地图</h3>
            <div class="view-toggle-panel">
              <button 
                @click="toggleView('map')" 
                :class="['view-toggle-btn', { active: !showVideoStream }]"
              >
                🗺️ 地图
              </button>
              <button 
                @click="toggleView('video')" 
                :class="['view-toggle-btn', { active: showVideoStream }]"
              >
                📹 视频
              </button>
            </div>
          </div>
          <div class="robot-filter">
            <div class="dropdown-container">
              <button @click="toggleDropdown" class="dropdown-btn">
                🤖 机器人筛选 
                <span :class="['dropdown-arrow', { open: showDropdown }]">▼</span>
              </button>
              <div v-show="showDropdown" class="dropdown-menu">
                <label v-for="robot in robots" :key="robot.id" class="dropdown-item">
                  <input 
                    type="checkbox" 
                    :value="robot.id" 
                    v-model="selectedRobots"
                  />
                  <span>{{ robot.id }}</span>
                </label>
              </div>
            </div>
          </div>
        </div>
        <div class="map-container">
          <!-- 地图背景 -->
          <div class="map-background" v-if="!showVideoStream">
            <div class="facility-layout">
              <div class="building">A栋</div>
              <div class="building">B栋</div>
              <div class="corridor">走廊</div>
            </div>
            
            <!-- AMG8833热力图 -->
            <div class="heatmap-overlay" v-if="showThermalOverlay">
              <!-- 热力图网格 -->
              <div 
                v-for="(row, rowIndex) in thermalGrid" 
                :key="'thermal-row-' + rowIndex"
                :style="{ display: 'grid', gridTemplateColumns: `repeat(${thermalGrid[0].length}, 1fr)`, height: `${100 / thermalGrid.length}%` }"
              >
                <div 
                  v-for="(temp, colIndex) in row"
                  :key="'thermal-cell-' + rowIndex + '-' + colIndex"
                  :style="{ 
                    backgroundColor: getThermalColor(temp),
                    opacity: getOpacityByTemp(temp)
                  }"
                  class="thermal-cell"
                ></div>
              </div>
            </div>
            
            <!-- 旧版热力点（可选显示） -->
            <div class="heatmap-overlay-old" v-if="!showThermalOverlay">
              <div 
                class="heat-point"
                v-for="(heatPoint, index) in heatPoints" 
                :key="'heat-'+index"
                :style="{ 
                  left: heatPoint.x + 'px', 
                  top: heatPoint.y + 'px',
                  backgroundColor: getHeatmapColor(heatPoint.temp),
                  width: heatPoint.radius + 'px',
                  height: heatPoint.radius + 'px'
                }"
              ></div>
            </div>
            
            <!-- 机器人位置 -->
            <div 
              class="robot-position"
              v-for="(robot, index) in visibleRobots"
              :key="index"
              :style="{ left: robot.x + 'px', top: robot.y + 'px' }"
            >
              <div class="robot-icon" :class="{ moving: robot.moving }">
                <i class="icon">🤖</i>
              </div>
              <div class="robot-info">
                <div class="robot-id">#{{ robot.id }}</div>
                <div class="robot-battery">电量: {{ robot.battery }}%</div>
              </div>
            </div>
          </div>
          
          <!-- 视频流显示 -->
          <div class="video-stream-container" v-if="showVideoStream">
            <div class="video-stream-content">
              <p class="video-title">机器人摄像头实时画面</p>
              <div class="video-stream" v-if="videoStreamActive">
                <img :src="currentVideoFrame" alt="实时视频流" />
              </div>
              <div class="no-video" v-else>
                <p>等待视频流连接...</p>
              </div>
            </div>
          </div>
        </div>
        
        <!-- 控制面板 -->
        <div class="control-panel" v-if="!showVideoStream">
          <label>
            <input type="checkbox" v-model="showThermalOverlay" /> 显示AMG8833 热力图
          </label>
          <button @click="toggleGridSize" class="btn btn-secondary">
            切换分辨率 ({{thermalGrid[0].length}}x{{thermalGrid.length}})
          </button>
        </div>
      </div>
      
      <!-- 右侧：数据概览和控制台 -->
      <div class="right-panel">
        <!-- 数据概览区域（缩小版） -->
        <div class="data-overview mini">
          <!-- 环境数据 -->
          <div class="data-section mini">
            <h4>环境监测</h4>
            <div class="environment-grid mini">
              <div class="env-item mini">
                <div class="env-label mini">温度</div>
                <div class="env-value mini">{{ environmentData.temperature }}°C</div>
              </div>
              <div class="env-item mini">
                <div class="env-label mini">湿度</div>
                <div class="env-value mini">{{ environmentData.humidity }}%</div>
              </div>
              <div class="env-item mini">
                <div class="env-label mini">气体浓度</div>
                <div class="env-value mini">{{ environmentData.gas }}ppm</div>
              </div>
              <div class="env-item mini">
                <div class="env-label mini">易燃气体</div>
                <div class="env-value mini">{{ environmentData.pm25 }}μg/m³</div>
              </div>
            </div>
          </div>

          <!-- 设备统计 -->
          <div class="data-section mini">
            <h4>设备统计</h4>
            <div class="stats-grid mini">
              <div class="stat-card mini">
                <div class="stat-icon bg-primary mini">
                  <i class="icon">🤖</i>
                </div>
                <div class="stat-info mini">
                  <h3>机器人总数</h3>
                  <p class="stat-number mini">{{ robotStats.total }}</p>
                  <p class="stat-change mini">+{{ robotStats.change }}</p>
                </div>
              </div>
              
              <div class="stat-card mini">
                <div class="stat-icon bg-success mini">
                  <i class="icon">🔋</i>
                </div>
                <div class="stat-info mini">
                  <h3>低电量</h3>
                  <p class="stat-number mini">{{ robotStats.lowBatteryCount }}</p>
                  <p class="stat-change mini"><20%</p>
                </div>
              </div>
              
              <div class="stat-card mini">
                <div class="stat-icon bg-warning mini">
                  <i class="icon">⚠️</i>
                </div>
                <div class="stat-info mini">
                  <h3>预警</h3>
                  <p class="stat-number mini">{{ alertStats.current }}</p>
                  <p class="stat-change mini">实时</p>
                </div>
              </div>
              
              <div class="stat-card mini">
                <div class="stat-icon bg-danger mini">
                  <i class="icon">🌐</i>
                </div>
                <div class="stat-info mini">
                  <h3>离线</h3>
                  <p class="stat-number mini">{{ robotStats.offline }}</p>
                  <p class="stat-change mini">{{ robotStats.offlineChange }}</p>
                </div>
              </div>
            </div>
          </div>
        </div>
        
        <!-- 控制台 -->
        <div class="activity-section card">
          <h3>控制台</h3>
          <ul class="activity-list">
            <li class="activity-item" v-for="(log, index) in mergedActivityLogs" :key="index" @click="scrollToMap">
              <div class="activity-icon">{{ log.icon }}</div>
              <div class="activity-content">
                <h4>{{ log.title }}</h4>
                <p v-html="formatLogMessage(log.message)"></p>
                <span class="activity-time">{{ log.time }}</span>
              </div>
            </li>
          </ul>
        </div>
      </div>
    </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, computed, watch } from 'vue';

// AMG8833 8x8 原始温度数据（最新数据）
const original_8x8 = [
  [16.8, 17.8, 18.2, 17.8, 16.8, 17.0, 14.0, 11.8],
  [17.0, 17.8, 18.2, 17.5, 18.5, 18.5, 17.5, 14.8],
  [16.0, 17.5, 17.8, 18.2, 18.0, 18.8, 19.2, 17.8],
  [16.8, 17.2, 17.8, 18.5, 18.8, 19.0, 18.2, 17.2],
  [16.5, 16.5, 16.8, 18.2, 19.2, 19.0, 18.0, 18.2],
  [15.2, 17.0, 16.8, 17.0, 18.8, 18.0, 18.2, 17.2],
  [14.2, 15.5, 15.5, 17.0, 17.5, 17.0, 15.2, 16.8],
  [13.2, 13.0, 12.8, 13.0, 15.0, 16.0, 14.2, 7.8]
];

// 32x32 插值温度数据（基于最新数据生成）
const heatData32x32 = [
  [16.8, 17.08, 17.35, 17.58, 17.74, 17.8, 17.74, 17.54, 17.19, 16.71, 16.15, 15.57, 15.04, 14.62, 14.36, 14.29, 14.41, 14.69, 15.08, 15.52, 15.94, 16.29, 16.53, 16.64, 16.63, 16.52, 16.34, 16.12, 15.9, 15.71, 15.58, 15.52],
  // ... 省略中间数据以保持简洁
  [23.31, 22.67, 22.01, 21.57, 21.32, 21.06, 21.23, 21.65, 22.21, 22.68, 23.04, 23.41, 23.79, 23.93, 24.0, 24.03, 24.34, 24.65, 24.68, 24.68, 24.38, 24.23, 24.23, 24.23, 24.19, 24.28, 24.63, 24.87, 24.95, 24.99, 24.84, 24.74]
];

// 机器人统计数据
const robotStats = ref({
  total: 12,
  change: 2,
  lowBatteryCount: 3, // 改为低电量设备数
  offline: 1, // 改为离线设备数
  offlineChange: '-1' // 离线设备变化
});

// 预警统计数据
const alertStats = ref({
  current: 3, // 改为当前预警设备数量
  week: 12
});

// 环境数据
const environmentData = ref({
  temperature: 23.5,
  humidity: 65,
  gas: 45,
  pm25: 28
});

// 机器人位置数据
const robots = ref([
  { id: 'R001', x: 150, y: 120, battery: 85, moving: true, visible: true },
  { id: 'R002', x: 280, y: 200, battery: 62, moving: false, visible: true },
  { id: 'R003', x: 420, y: 100, battery: 95, moving: true, visible: true }
]);

// 选中的机器人 ID 列表（用于下拉多选）
const selectedRobots = ref(['R001', 'R002', 'R003']);

// 计算属性：只显示选中的机器人
const visibleRobots = computed(() => {
  return robots.value.filter(robot => selectedRobots.value.includes(robot.id));
});

// 监听选中状态变化，同步更新机器人的 visible 属性
watch(selectedRobots, (newSelected) => {
  robots.value.forEach(robot => {
    robot.visible = newSelected.includes(robot.id);
  });
}, { deep: true });

// 切换机器人显示状态
const toggleRobotVisibility = (robotId) => {
  const robot = robots.value.find(r => r.id === robotId);
  if (robot) {
    robot.visible = !robot.visible;
  }
};

// 热力点数据（用于旧版热力图）
const heatPoints = ref([
  { x: 100, y: 80, temp: 20, radius: 30 },
  { x: 200, y: 150, temp: 32, radius: 25 },
  { x: 350, y: 180, temp: 42, radius: 40 },
  { x: 450, y: 90, temp: 28, radius: 20 },
  { x: 280, y: 250, temp: 48, radius: 45 }
]);

// 新增：AMG8833热成像数据相关
const showThermalOverlay = ref(true);  // 是否显示热力图叠加层
const gridSize = ref(32);  // 默认32x32网格
let thermalGrid = ref(heatData32x32);  // 当前热成像网格数据

// 视频流数据
const videoStreamActive = ref(true);
const currentVideoFrame = ref(''); // 实际应用中会是视频帧的 URL 或 base64 数据
const showVideoStream = ref(false); // 控制是否显示视频流
const showDropdown = ref(false); // 控制下拉菜单显示

// 切换视图
const toggleView = (view) => {
  showVideoStream.value = (view === 'video');
};

// 切换下拉菜单显示
const toggleDropdown = () => {
  showDropdown.value = !showDropdown.value;
};

// 活动日志
const activityLogs = ref([
  { icon: '🔋', title: '电量提醒', message: '机器人 #R002 电量低于20%', time: '2分钟前' },
  { icon: '🔋', title: '电量提醒', message: '机器人 #R004 电量低于15%', time: '5分钟前' },
  { icon: '⚠️', title: '预警信息', message: 'B栋检测到温度异常', time: '5分钟前' },
  { icon: '⚠️', title: '预警信息', message: 'A栋检测到烟雾浓度过高', time: '8分钟前' },
  { icon: '📍', title: '位置更新', message: '机器人 #R001 到达A栋大厅', time: '8分钟前' },
  { icon: '📍', title: '位置更新', message: '机器人 #R003 到达B栋走廊', time: '10分钟前' },
  { icon: '✅', title: '任务完成', message: '机器人 #R002 完成巡检任务', time: '15分钟前' },
  { icon: '✅', title: '任务完成', message: '机器人 #R005 完成夜间巡检', time: '20分钟前' }
]);

// 计算属性：合并相同title的活动日志
const mergedActivityLogs = computed(() => {
  const merged = {};
  
  activityLogs.value.forEach(log => {
    if (!merged[log.title]) {
      merged[log.title] = {
        icon: log.icon,
        title: log.title,
        messages: [],
        latestTime: log.time
      };
    }
    
    // 添加消息
    merged[log.title].messages.push(log.message);
    
    // 更新最新时间
    if (new Date(log.time) > new Date(merged[log.title].latestTime)) {
      merged[log.title].latestTime = log.time;
    }
  });
  
  // 转换为数组并按时间排序
  return Object.values(merged)
    .map(item => ({
      icon: item.icon,
      title: item.title,
      message: item.messages, // 保持为数组格式
      time: item.latestTime
    }))
    .sort((a, b) => new Date(b.time) - new Date(a.time));
});

// 格式化日志消息，将数组转换为分行显示
const formatLogMessage = (messages) => {
  if (Array.isArray(messages)) {
    return messages.join('<br>');
  }
  return messages;
};

const scrollToMap = () => {
  const mapSection = document.querySelector('.map-section');
  mapSection.scrollIntoView({ behavior: 'smooth' });
};

const getHeatmapColor = (temperature) => {
  // 根据温度值返回不同的颜色，符合风险热力图颜色编码规范
  if (temperature < 20) return 'rgba(33, 150, 243, 0.6)'; // 蓝色 - 低温
  if (temperature <= 30) return 'rgba(33, 150, 243, 0.8)'; // 浅蓝
  if (temperature <= 35) return 'rgba(255, 193, 7, 0.6)'; // 黄色 - 中温
  if (temperature <= 40) return 'rgba(255, 152, 0, 0.7)'; // 橙色
  return 'rgba(244, 67, 54, 0.6)'; // 红色 - 高温
};

// 新增：根据温度值获取热力图颜色（蓝-黄-红）
const getThermalColor = (temp) => {
  // 根据温度值返回对应的颜色，使用蓝-黄-红渐变
  const minTemp = 7.8; // 最低温度（根据最新数据调整）
  const maxTemp = 24.68; // 最高温度（根据最新数据调整）
  const ratio = Math.min(Math.max((temp - minTemp) / (maxTemp - minTemp), 0), 1);

  let r, g, b;
  if (ratio < 0.5) {
    // 从蓝色到黄色 (0.0 to 0.5)
    const t = ratio * 2;
    r = Math.floor(255 * t);
    g = Math.floor(255 * Math.min(t * 2, 1));
    b = Math.floor(255 * (1 - t));
  } else {
    // 从黄色到红色 (0.5 to 1.0)
    const t = (ratio - 0.5) * 2;
    r = 255;
    g = Math.floor(255 * (1 - t));
    b = 0;
  }

  return `rgb(${r}, ${g}, ${b})`;
};

// 新增：根据温度值计算透明度
const getOpacityByTemp = (temp) => {
  // 温度越高，透明度越低（更明显）
  const minTemp = 7.8; // 最低温度
  const maxTemp = 24.68; // 最高温度
  const ratio = Math.min(Math.max((temp - minTemp) / (maxTemp - minTemp), 0), 1);
  return 0.3 + ratio * 0.7; // 0.3 to 1.0
};

// 新增：切换网格大小
const toggleGridSize = () => {
  if (gridSize.value === 32) {
    gridSize.value = 8;
    thermalGrid.value = original_8x8;
  } else {
    gridSize.value = 32;
    thermalGrid.value = heatData32x32;
  }
};

// SSE连接
let eventSource = null;

// 初始化SSE连接
const initSSEConnection = () => {
  if (eventSource) {
    eventSource.close();
  }

  // 创建 SSE连接，使用/sse/stream 端点
  eventSource = new EventSource('/api/sse/stream');

  // 监听消息事件
  eventSource.onmessage = (event) => {
    try {
      const data = JSON.parse(event.data);
      updateDashboardData(data);
    } catch (error) {
      console.error('解析 SSE 数据失败:', error);
    }
  };

  // 监听错误事件
  eventSource.onerror = (error) => {
    console.error('SSE连接错误:', error);
    // 尝试重新连接
    eventSource.close();
    setTimeout(initSSEConnection, 5000);
  };

  // 监听连接打开事件
  eventSource.onopen = () => {
    console.log('SSE连接已建立 - /api/sse/stream');
  };
};

// 更新仪表板数据
const updateDashboardData = (data) => {
  // 更新机器人统计数据
  if (data.robotStats) {
    robotStats.value = {
      total: data.robotStats.total || robotStats.value.total,
      change: data.robotStats.change || robotStats.value.change,
      lowBatteryCount: data.robotStats.lowBatteryCount || 0,
      offline: data.robotStats.offline || 0,
      offlineChange: data.robotStats.offlineChange || robotStats.value.offlineChange
    };
  }

  // 更新预警统计数据
  if (data.alertStats) {
    alertStats.value = {
      current: data.alertStats.current || 0,
      week: data.alertStats.week || alertStats.value.week
    };
  }

  // 更新环境数据
  if (data.environment) {
    environmentData.value = {
      temperature: data.environment.temperature || environmentData.value.temperature,
      humidity: data.environment.humidity || environmentData.value.humidity,
      gas: data.environment.gas || environmentData.value.gas,
      pm25: data.environment.pm25 || environmentData.value.pm25
    };
  }

  // 更新机器人位置
  if (data.robots && Array.isArray(data.robots)) {
    robots.value = data.robots.map(robot => ({
      id: robot.id,
      x: robot.x || Math.random() * 400 + 50,
      y: robot.y || Math.random() * 300 + 50,
      battery: robot.battery || Math.floor(Math.random() * 100),
      moving: robot.moving || Math.random() > 0.5
    }));
  }

  // 更新热力点数据
  if (data.heatPoints && Array.isArray(data.heatPoints)) {
    heatPoints.value = data.heatPoints.map(point => ({
      x: point.x || Math.random() * 500,
      y: point.y || Math.random() * 350,
      temp: point.temp || Math.random() * 30 + 10,
      radius: point.radius || Math.random() * 30 + 10
    }));
  }

  // 更新活动日志
  if (data.activityLogs && Array.isArray(data.activityLogs)) {
    // 只保留最新的10条日志
    const newLogs = data.activityLogs.slice(0, 10);
    
    // 如果日志内容有变化，才更新
    if (JSON.stringify(activityLogs.value) !== JSON.stringify(newLogs)) {
      activityLogs.value = newLogs;
    }
  }

  // 更新热成像数据
  if (data.thermalData) {
    if (data.thermalData.original_8x8) {
      original_8x8.splice(0, original_8x8.length, ...data.thermalData.original_8x8);
    }
    if (data.thermalData.heatData32x32) {
      thermalGrid.value = data.thermalData.heatData32x32;
    }
  }
};

onMounted(() => {
  // 初始化SSE连接
  initSSEConnection();
  
  // 初始加载数据，使用/sse/stream 端点
  fetch('/api/sse/stream')
    .then(response => response.json())
    .then(data => {
      updateDashboardData(data);
    })
    .catch(error => {
      console.error('初始数据加载失败:', error);
    });
});

onUnmounted(() => {
  // 清理SSE连接
  if (eventSource) {
    eventSource.close();
    eventSource = null;
  }
});
</script>

<style scoped>
/* 全局样式 - 确保页面 100% 高度 */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html, body {
  width: 100%;
  height: 100%;
  overflow: hidden;
}

#app {
  width: 100%;
  height: 100%;
}

/* 整页布局样式 */
.dashboard-container {
  padding: 0;
  width: 100%;
  height: 100vh;
  margin: 0;
  background: #f5f7fa;
  overflow-x: hidden;
  overflow-y: auto;
}

.page-title {
  padding: 6px;
  margin-bottom: 6px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  text-align: center;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
  flex-shrink: 0;
}

/* 其他内容区域样式 - 紧凑 */
.content-grid {
  width: 100%;
  display: grid;
  grid-template-columns: 2fr 1fr;
  gap: 12px;
  padding: 0 12px 12px 12px;
  box-sizing: border-box;
  flex: 1;
  overflow: hidden;
}

.map-section, .video-section, .activity-section {
  width: 100%;
  padding: 12px;
  background: white;
  border-radius: 8px;
  box-shadow: 0 3px 10px rgba(0,0,0,0.12);
  box-sizing: border-box;
}

/* 数据概览区域 - 缩小版 */
.data-overview.mini {
  width: 100%;
  padding: 10px;
  margin: 0 0 12px 0;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.data-section.mini {
  width: 100%;
  margin-bottom: 8px;
}

.data-section.mini:last-child {
  margin-bottom: 0;
}

.data-section.mini h4 {
  margin: 0 0 3px 0;
  color: #2c3e50;
  font-size: 0.75rem;
  font-weight: 600;
  text-align: center;
}

.environment-grid.mini {
  width: 100%;
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 6px;
  margin-bottom: 8px;
}

.environment-grid.mini:last-child {
  margin-bottom: 0;
}

.env-item.mini {
  width: 100%;
  padding: 8px;
  background: #f8f9fa;
  border-radius: 4px;
  text-align: center;
  border: 1px solid #e9ecef;
  box-sizing: border-box;
}

.env-label.mini {
  font-size: 0.7rem;
  color: #7f8c8d;
  margin-bottom: 4px;
}

.env-value.mini {
  font-size: 0.9rem;
  font-weight: bold;
  color: #2c3e50;
}

.stats-grid.mini {
  width: 100%;
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 6px;
}

.stat-card.mini {
  width: 100%;
  display: flex;
  padding: 8px;
  background: white;
  border-radius: 6px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.06);
  border: 1px solid #e9ecef;
  box-sizing: border-box;
}

.stat-icon.mini {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border-radius: 4px;
  margin-right: 6px;
  font-size: 0.9rem;
  flex-shrink: 0;
}

.stat-info.mini h3 {
  font-size: 0.65rem;
  color: #7f8c8d;
  margin-bottom: 2px;
  line-height: 1.2;
}

.stat-number.mini {
  font-size: 0.95rem;
  font-weight: 700;
  color: #2c3e50;
  margin-bottom: 2px;
}

.stat-change.mini {
  font-size: 0.6rem;
  font-weight: 500;
}

.map-section, .video-section, .activity-section {
  width: 100%;
  padding: 5px;
  background: white;
  border-radius: 4px;
  box-shadow: 0 1px 5px rgba(0,0,0,0.08);
  box-sizing: border-box;
}

.map-section h3, .video-section h3, .activity-section h3 {
  margin-bottom: 5px;
  color: #2c3e50;
  font-size: 1rem;
}

/* 地图头部布局 */
.map-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}

.map-title-group {
  display: flex;
  align-items: center;
  gap: 12px;
}

.map-title-group h3 {
  margin: 0;
  color: #2c3e50;
  font-size: 1rem;
}

/* 视图切换按钮面板 */
.view-toggle-panel {
  display: flex;
  gap: 4px;
}

.view-toggle-btn {
  padding: 3px 8px;
  font-size: 0.7rem;
  background: #f0f5ff;
  border: 1px solid #d1e0f0;
  border-radius: 3px;
  cursor: pointer;
  transition: all 0.2s;
  color: #4a6d8d;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 3px;
}

.view-toggle-btn:hover {
  background: #e0eaf5;
  border-color: #a4c4e0;
}

.view-toggle-btn.active {
  background: #2575fc;
  color: white;
  border-color: #1a68f5;
}

/* 机器人筛选下拉菜单容器 */
.robot-filter {
  display: flex;
  gap: 4px;
}

.dropdown-container {
  position: relative;
}

.dropdown-btn {
  padding: 6px 12px;
  font-size: 0.75rem;
  background: #f0f5ff;
  border: 1px solid #d1e0f0;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s;
  color: #4a6d8d;
  display: flex;
  align-items: center;
  gap: 6px;
  font-weight: 500;
}

.dropdown-btn:hover {
  background: #e0eaf5;
  border-color: #a4c4e0;
}

.dropdown-arrow {
  font-size: 0.6rem;
  transition: transform 0.3s;
}

.dropdown-arrow.open {
  transform: rotate(180deg);
}

.dropdown-menu {
  position: absolute;
  top: 100%;
  right: 0;
  margin-top: 6px;
  background: white;
  border: 1px solid #d1e0f0;
  border-radius: 6px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.15);
  padding: 8px;
  min-width: 140px;
  z-index: 1000;
  animation: slideDown 0.2s ease-out;
}

@keyframes slideDown {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.dropdown-item {
  display: flex;
  align-items: center;
  padding: 6px 8px;
  cursor: pointer;
  transition: background-color 0.2s;
  border-radius: 4px;
  gap: 8px;
}

.dropdown-item:hover {
  background-color: #f0f5ff;
}

.dropdown-item input[type="checkbox"] {
  width: 16px;
  height: 16px;
  cursor: pointer;
  accent-color: #2575fc;
}

.dropdown-item span {
  font-size: 0.8rem;
  color: #2c3e50;
  font-weight: 500;
}

.map-section {
  display: flex;
  flex-direction: column;
  height: 100%;
}

.map-container {
  position: relative;
  width: 100%;
  flex: 1;
  min-height: 500px;
  overflow: hidden;
  background: #f0f5ff;
  border-radius: 8px;
}

.map-background {
  position: relative;
  width: 100%;
  height: 100%;
  background: #e6f0ff;
}

.facility-layout {
  position: absolute;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: space-around;
  align-items: center;
  padding: 6px;
  box-sizing: border-box;
}

.building {
  width: 50px;
  height: 50px;
  background: #d1e0f0;
  border: 1px solid #a4c4e0;
  border-radius: 3px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #4a6d8d;
  font-weight: bold;
  font-size: 0.7rem;
}

.corridor {
  width: 70px;
  height: 25px;
  background: #f0f5ff;
  border: 1px dashed #a4c4e0;
  border-radius: 2px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #7f8c8d;
  font-size: 0.6rem;
}

.robot-position {
  position: absolute;
  display: flex;
  flex-direction: column;
  align-items: center;
  transition: all 0.5s ease;
}

.robot-icon {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  background: #fff;
  border: 1px solid #2575fc;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2px 4px rgba(0,0,0,0.2);
  transition: transform 0.3s;
}

.robot-icon.moving {
  animation: pulse 1.5s infinite;
}

@keyframes pulse {
  0% { transform: scale(1); }
  50% { transform: scale(1.1); }
  100% { transform: scale(1); }
}

.robot-info {
  background: rgba(255, 255, 255, 0.9);
  border-radius: 2px;
  padding: 2px 4px;
  margin-top: 3px;
  font-size: 0.55rem;
  box-shadow: 0 1px 2px rgba(0,0,0,0.1);
}

.robot-id {
  font-weight: bold;
  color: #2c3e50;
}

.robot-battery {
  color: #7f8c8d;
}

.heat-point {
  position: absolute;
  border-radius: 50%;
  opacity: 0.7;
  transform: translate(-50%, -50%);
  border: 1px solid rgba(0, 0, 0, 0.2);
}

.heatmap-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
}

.heatmap-overlay-old {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
}

.thermal-cell {
  border: 1px dotted transparent;
  transition: opacity 0.3s ease;
}

/* 视图切换按钮面板 */
.view-toggle-panel {
  margin-top: 5px;
  display: flex;
  gap: 5px;
  justify-content: center;
}

.view-toggle-btn {
  flex: 1;
  padding: 6px 12px;
  font-size: 0.75rem;
  background: #f0f5ff;
  border: 1px solid #d1e0f0;
  border-radius: 3px;
  cursor: pointer;
  transition: all 0.2s;
  color: #4a6d8d;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 4px;
}

.view-toggle-btn:hover {
  background: #e0eaf5;
  border-color: #a4c4e0;
}

.view-toggle-btn.active {
  background: #2575fc;
  color: white;
  border-color: #1a68f5;
}

/* 视频流容器 */
.video-stream-container {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: #f0f5ff;
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.video-stream-content {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
}

.video-title {
  text-align: center;
  padding: 4px;
  background: rgba(44, 62, 80, 0.9);
  color: white;
  font-size: 0.65rem;
  font-weight: 600;
}

.video-stream {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #000;
}

.video-stream img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.no-video {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #7f8c8d;
  background: #2c3e50;
}

.control-panel {
  margin-top: 12px;
  display: flex;
  justify-content: space-around;
  align-items: center;
  padding: 10px;
  background: #f8f9fa;
  border-radius: 8px;
}

.btn {
  padding: 4px 8px;
  border: 1px solid #ddd;
  border-radius: 2px;
  background: #fff;
  cursor: pointer;
  font-size: 0.7rem;
}

.btn-secondary {
  background: #6c757d;
  color: white;
  border: 1px solid #6c757d;
}

.video-container {
  width: 100%;
  height: 120px;
  background: #000;
  border-radius: 4px;
  overflow: hidden;
}

.video-stream {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
}

.video-stream img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.no-video {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #7f8c8d;
  background: #2c3e50;
}

/* 右侧面板布局 */
.right-panel {
  display: flex;
  flex-direction: column;
  gap: 5px;
  width: 100%;
  height: 100%;
  overflow: hidden;
}

.data-overview.mini {
  flex-shrink: 0;
}

.activity-section {
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;
}

.activity-section h3 {
  flex-shrink: 0;
  margin-bottom: 12px;
  font-size: 1.1rem;
}

.activity-list {
  flex: 1;
  min-height: 0;
  max-height: none;
  overflow-y: auto;
}

.activity-item {
  display: flex;
  padding: 8px 0;
  border-bottom: 1px solid #eee;
  cursor: pointer;
  transition: background-color 0.2s;
}

.activity-item:last-child {
  border-bottom: none;
}

.activity-item:hover {
  background-color: #f8f9fa;
}

.activity-icon {
  font-size: 1.1rem;
  margin-right: 6px;
  width: 28px;
  height: 28px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #f8f9fa;
  border-radius: 6px;
}

.activity-content {
  flex: 1;
}

.activity-content h4 {
  margin: 0 0 3px 0;
  color: #2c3e50;
  font-size: 0.75rem;
}

.activity-content p {
  margin: 0 0 3px 0;
  color: #7f8c8d;
  font-size: 0.65rem;
  line-height: 1.2;
}

.activity-time {
  font-size: 0.55rem;
  color: #95a5a6;
}

/* 响应式设计 */
@media (max-width: 768px) {
  .content-grid {
    grid-template-columns: 1fr;
    gap: 4px;
  }
  
  .data-overview {
    margin: 4px;
    padding: 4px;
    margin-bottom: 6px;
  }
  
  .data-overview.compact {
    margin: 4px;
    padding: 4px;
    margin-bottom: 6px;
  }
  
  .environment-grid {
    grid-template-columns: repeat(2, 1fr);
  }
  
  .environment-grid.compact {
    gap: 3px;
  }
  
  .stats-grid {
    grid-template-columns: 1fr;
  }
  
  .stats-grid.compact {
    gap: 3px;
  }
  
  .stat-card {
    padding: 4px;
  }
  
  .map-container {
    height: 200px;
  }
  
  .video-container {
    height: 100px;
  }
}

@media (max-width: 480px) {
  .environment-grid {
    grid-template-columns: 1fr;
  }
  
  .stat-card {
    flex-direction: column;
    text-align: center;
  }
  
  .stat-icon {
    margin-right: 0;
    margin-bottom: 3px;
  }
  
  .map-section, .video-section, .activity-section {
    margin: 0;
    padding: 4px;
  }
  
  .page-title {
    font-size: 1rem;
    padding: 4px;
    margin-bottom: 4px;
  }
  
  .map-container {
    height: 180px;
  }
  
  .building {
    width: 45px;
    height: 45px;
    font-size: 0.6rem;
  }
  
  .robot-icon {
    width: 24px;
    height: 24px;
  }
}
</style>