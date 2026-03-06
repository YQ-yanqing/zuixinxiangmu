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
              <div class="building">A 栋</div>
              <div class="building">B 栋</div>
              <div class="corridor">走廊</div>
            </div>
            
            <!-- AMG8833 热力图 -->
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
                <div class="robot-battery">电量：{{ robot.battery }}%</div>
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
            <input type="checkbox" v-model="showThermalOverlay" /> 显示 AMG8833 热力图
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
        
        <!-- 温度折线图（替换原控制台） -->
        <div class="activity-section card">
          <h3>温度趋势</h3>
          <div ref="temperatureChart" class="chart-container"></div>
        </div>
      </div>
    </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, computed, watch } from 'vue';
import * as echarts from 'echarts';

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
  [16.85, 17.13, 17.41, 17.64, 17.81, 17.87, 17.81, 17.61, 17.26, 16.78, 16.22, 15.64, 15.11, 14.69, 14.43, 14.36, 14.48, 14.76, 15.15, 15.59, 16.01, 16.36, 16.6, 16.71, 16.7, 16.59, 16.41, 16.19, 15.97, 15.78, 15.65, 15.59],
  [17.0, 17.28, 17.56, 17.79, 17.96, 18.02, 17.96, 17.76, 17.41, 16.93, 16.37, 15.79, 15.26, 14.84, 14.58, 14.51, 14.63, 14.91, 15.3, 15.74, 16.16, 16.51, 16.75, 16.86, 16.85, 16.74, 16.56, 16.34, 16.12, 15.93, 15.8, 15.74],
  [17.15, 17.43, 17.71, 17.94, 18.11, 18.17, 18.11, 17.91, 17.56, 17.08, 16.52, 15.94, 15.41, 14.99, 14.73, 14.66, 14.78, 15.06, 15.45, 15.89, 16.31, 16.66, 16.9, 17.01, 17.0, 16.89, 16.71, 16.49, 16.27, 16.08, 15.95, 15.89],
  [17.3, 17.58, 17.86, 18.09, 18.26, 18.32, 18.26, 18.06, 17.71, 17.23, 16.67, 16.09, 15.56, 15.14, 14.88, 14.81, 14.93, 15.21, 15.6, 16.04, 16.46, 16.81, 17.05, 17.16, 17.15, 17.04, 16.86, 16.64, 16.42, 16.23, 16.1, 16.04],
  [17.45, 17.73, 18.01, 18.24, 18.41, 18.47, 18.41, 18.21, 17.86, 17.38, 16.82, 16.24, 15.71, 15.29, 15.03, 14.96, 15.08, 15.36, 15.75, 16.19, 16.61, 16.96, 17.2, 17.31, 17.3, 17.19, 17.01, 16.79, 16.57, 16.38, 16.25, 16.19],
  [17.6, 17.88, 18.16, 18.39, 18.56, 18.62, 18.56, 18.36, 18.01, 17.53, 16.97, 16.39, 15.86, 15.44, 15.18, 15.11, 15.23, 15.51, 15.9, 16.34, 16.76, 17.11, 17.35, 17.46, 17.45, 17.34, 17.16, 16.94, 16.72, 16.53, 16.4, 16.34],
  [17.75, 18.03, 18.31, 18.54, 18.71, 18.77, 18.71, 18.51, 18.16, 17.68, 17.12, 16.54, 16.01, 15.59, 15.33, 15.26, 15.38, 15.66, 16.05, 16.49, 16.91, 17.26, 17.5, 17.61, 17.6, 17.49, 17.31, 17.09, 16.87, 16.68, 16.55, 16.49],
  [17.9, 18.18, 18.46, 18.69, 18.86, 18.92, 18.86, 18.66, 18.31, 17.83, 17.27, 16.69, 16.16, 15.74, 15.48, 15.41, 15.53, 15.81, 16.2, 16.64, 17.06, 17.41, 17.65, 17.76, 17.75, 17.64, 17.46, 17.24, 17.02, 16.83, 16.7, 16.64],
  [18.05, 18.33, 18.61, 18.84, 19.01, 19.07, 19.01, 18.81, 18.46, 17.98, 17.42, 16.84, 16.31, 15.89, 15.63, 15.56, 15.68, 15.96, 16.35, 16.79, 17.21, 17.56, 17.8, 17.91, 17.9, 17.79, 17.61, 17.39, 17.17, 16.98, 16.85, 16.79],
  [18.2, 18.48, 18.76, 18.99, 19.16, 19.22, 19.16, 18.96, 18.61, 18.13, 17.57, 16.99, 16.46, 16.04, 15.78, 15.71, 15.83, 16.11, 16.5, 16.94, 17.36, 17.71, 17.95, 18.06, 18.05, 17.94, 17.76, 17.54, 17.32, 17.13, 17.0, 16.94],
  [18.35, 18.63, 18.91, 19.14, 19.31, 19.37, 19.31, 19.11, 18.76, 18.28, 17.72, 17.14, 16.61, 16.19, 15.93, 15.86, 15.98, 16.26, 16.65, 17.09, 17.51, 17.86, 18.1, 18.21, 18.2, 18.09, 17.91, 17.69, 17.47, 17.28, 17.15, 17.09],
  [18.5, 18.78, 19.06, 19.29, 19.46, 19.52, 19.46, 19.26, 18.91, 18.43, 17.87, 17.29, 16.76, 16.34, 16.08, 16.01, 16.13, 16.41, 16.8, 17.24, 17.66, 18.01, 18.25, 18.36, 18.35, 18.24, 18.06, 17.84, 17.62, 17.43, 17.3, 17.24],
  [18.65, 18.93, 19.21, 19.44, 19.61, 19.67, 19.61, 19.41, 19.06, 18.58, 18.02, 17.44, 16.91, 16.49, 16.23, 16.16, 16.28, 16.56, 16.95, 17.39, 17.81, 18.16, 18.4, 18.51, 18.5, 18.39, 18.21, 17.99, 17.77, 17.58, 17.45, 17.39],
  [18.8, 19.08, 19.36, 19.59, 19.76, 19.82, 19.76, 19.56, 19.21, 18.73, 18.17, 17.59, 17.06, 16.64, 16.38, 16.31, 16.43, 16.71, 17.1, 17.54, 17.96, 18.31, 18.55, 18.66, 18.65, 18.54, 18.36, 18.14, 17.92, 17.73, 17.6, 17.54],
  [18.95, 19.23, 19.51, 19.74, 19.91, 19.97, 19.91, 19.71, 19.36, 18.88, 18.32, 17.74, 17.21, 16.79, 16.53, 16.46, 16.58, 16.86, 17.25, 17.69, 18.11, 18.46, 18.7, 18.81, 18.8, 18.69, 18.51, 18.29, 18.07, 17.88, 17.75, 17.69],
  [19.1, 19.38, 19.66, 19.89, 20.06, 20.12, 20.06, 19.86, 19.51, 19.03, 18.47, 17.89, 17.36, 16.94, 16.68, 16.61, 16.73, 17.01, 17.4, 17.84, 18.26, 18.61, 18.85, 18.96, 18.95, 18.84, 18.66, 18.44, 18.22, 18.03, 17.9, 17.84],
  [19.25, 19.53, 19.81, 20.04, 20.21, 20.27, 20.21, 20.01, 19.66, 19.18, 18.62, 18.04, 17.51, 17.09, 16.83, 16.76, 16.88, 17.16, 17.55, 17.99, 18.41, 18.76, 19.0, 19.11, 19.1, 18.99, 18.81, 18.59, 18.37, 18.18, 18.05, 17.99],
  [19.4, 19.68, 19.96, 20.19, 20.36, 20.42, 20.36, 20.16, 19.81, 19.33, 18.77, 18.19, 17.66, 17.24, 16.98, 16.91, 17.03, 17.31, 17.7, 18.14, 18.56, 18.91, 19.15, 19.26, 19.25, 19.14, 18.96, 18.74, 18.52, 18.33, 18.2, 18.14],
  [19.55, 19.83, 20.11, 20.34, 20.51, 20.57, 20.51, 20.31, 19.96, 19.48, 18.92, 18.34, 17.81, 17.39, 17.13, 17.06, 17.18, 17.46, 17.85, 18.29, 18.71, 19.06, 19.3, 19.41, 19.4, 19.29, 19.11, 18.89, 18.67, 18.48, 18.35, 18.29],
  [19.7, 19.98, 20.26, 20.49, 20.66, 20.72, 20.66, 20.46, 20.11, 19.63, 19.07, 18.49, 17.96, 17.54, 17.28, 17.21, 17.33, 17.61, 18.0, 18.44, 18.86, 19.21, 19.45, 19.56, 19.55, 19.44, 19.26, 19.04, 18.82, 18.63, 18.5, 18.44],
  [19.85, 20.13, 20.41, 20.64, 20.81, 20.87, 20.81, 20.61, 20.26, 19.78, 19.22, 18.64, 18.11, 17.69, 17.43, 17.36, 17.48, 17.76, 18.15, 18.59, 19.01, 19.36, 19.6, 19.71, 19.7, 19.59, 19.41, 19.19, 18.97, 18.78, 18.65, 18.59],
  [20.0, 20.28, 20.56, 20.79, 20.96, 21.02, 20.96, 20.76, 20.41, 19.93, 19.37, 18.79, 18.26, 17.84, 17.58, 17.51, 17.63, 17.91, 18.3, 18.74, 19.16, 19.51, 19.75, 19.86, 19.85, 19.74, 19.56, 19.34, 19.12, 18.93, 18.8, 18.74],
  [20.15, 20.43, 20.71, 20.94, 21.11, 21.17, 21.11, 20.91, 20.56, 20.08, 19.52, 18.94, 18.41, 17.99, 17.73, 17.66, 17.78, 18.06, 18.45, 18.89, 19.31, 19.66, 19.9, 20.01, 20.0, 19.89, 19.71, 19.49, 19.27, 19.08, 18.95, 18.89],
  [20.3, 20.58, 20.86, 21.09, 21.26, 21.32, 21.26, 21.06, 20.71, 20.23, 19.67, 19.09, 18.56, 18.14, 17.88, 17.81, 17.93, 18.21, 18.6, 19.04, 19.46, 19.81, 20.05, 20.16, 20.15, 20.04, 19.86, 19.64, 19.42, 19.23, 19.1, 19.04],
  [20.45, 20.73, 21.01, 21.24, 21.41, 21.47, 21.41, 21.21, 20.86, 20.38, 19.82, 19.24, 18.71, 18.29, 18.03, 17.96, 18.08, 18.36, 18.75, 19.19, 19.61, 19.96, 20.2, 20.31, 20.3, 20.19, 20.01, 19.79, 19.57, 19.38, 19.25, 19.19],
  [20.6, 20.88, 21.16, 21.39, 21.56, 21.62, 21.56, 21.36, 21.01, 20.53, 19.97, 19.39, 18.86, 18.44, 18.18, 18.11, 18.23, 18.51, 18.9, 19.34, 19.76, 20.11, 20.35, 20.46, 20.45, 20.34, 20.16, 19.94, 19.72, 19.53, 19.4, 19.34],
  [20.75, 21.03, 21.31, 21.54, 21.71, 21.77, 21.71, 21.51, 21.16, 20.68, 20.12, 19.54, 19.01, 18.59, 18.33, 18.26, 18.38, 18.66, 19.05, 19.49, 19.91, 20.26, 20.5, 20.61, 20.6, 20.49, 20.31, 20.09, 19.87, 19.68, 19.55, 19.49],
  [20.9, 21.18, 21.46, 21.69, 21.86, 21.92, 21.86, 21.66, 21.31, 20.83, 20.27, 19.69, 19.16, 18.74, 18.48, 18.41, 18.53, 18.81, 19.2, 19.64, 20.06, 20.41, 20.65, 20.76, 20.75, 20.64, 20.46, 20.24, 20.02, 19.83, 19.7, 19.64],
  [21.05, 21.33, 21.61, 21.84, 22.01, 22.07, 22.01, 21.81, 21.46, 20.98, 20.42, 19.84, 19.31, 18.89, 18.63, 18.56, 18.68, 18.96, 19.35, 19.79, 20.21, 20.56, 20.8, 20.91, 20.9, 20.79, 20.61, 20.39, 20.17, 19.98, 19.85, 19.79],
  [21.2, 21.48, 21.76, 21.99, 22.16, 22.22, 22.16, 21.96, 21.61, 21.13, 20.57, 19.99, 19.46, 19.04, 18.78, 18.71, 18.83, 19.11, 19.5, 19.94, 20.36, 20.71, 20.95, 21.06, 21.05, 20.94, 20.76, 20.54, 20.32, 20.13, 20.0, 19.94],
  [21.35, 21.63, 21.91, 22.14, 22.31, 22.37, 22.31, 22.11, 21.76, 21.28, 20.72, 20.14, 19.61, 19.19, 18.93, 18.86, 18.98, 19.26, 19.65, 20.09, 20.51, 20.86, 21.1, 21.21, 21.2, 21.09, 20.91, 20.69, 20.47, 20.28, 20.15, 20.09],
  [21.5, 21.78, 22.06, 22.29, 22.46, 22.52, 22.46, 22.26, 21.91, 21.43, 20.87, 20.29, 19.76, 19.34, 19.08, 19.01, 19.13, 19.41, 19.8, 20.24, 20.66, 21.01, 21.25, 21.36, 21.35, 21.24, 21.06, 20.84, 20.62, 20.43, 20.3, 20.24],
  [21.65, 21.93, 22.21, 22.44, 22.61, 22.67, 22.61, 22.41, 22.06, 21.58, 21.02, 20.44, 19.91, 19.49, 19.23, 19.16, 19.28, 19.56, 19.95, 20.39, 20.81, 21.16, 21.4, 21.51, 21.5, 21.39, 21.21, 20.99, 20.77, 20.58, 20.45, 20.39],
  [21.8, 22.08, 22.36, 22.59, 22.76, 22.82, 22.76, 22.56, 22.21, 21.73, 21.17, 20.59, 20.06, 19.64, 19.38, 19.31, 19.43, 19.71, 20.1, 20.54, 20.96, 21.31, 21.55, 21.66, 21.65, 21.54, 21.36, 21.14, 20.92, 20.73, 20.6, 20.54],
  [21.95, 22.23, 22.51, 22.74, 22.91, 22.97, 22.91, 22.71, 22.36, 21.88, 21.32, 20.74, 20.21, 19.79, 19.53, 19.46, 19.58, 19.86, 20.25, 20.69, 21.11, 21.46, 21.7, 21.81, 21.8, 21.69, 21.51, 21.29, 21.07, 20.88, 20.75, 20.69],
  [22.1, 22.38, 22.66, 22.89, 23.06, 23.12, 23.06, 22.86, 22.51, 22.03, 21.47, 20.89, 20.36, 19.94, 19.68, 19.61, 19.73, 20.01, 20.4, 20.84, 21.26, 21.61, 21.85, 21.96, 21.95, 21.84, 21.66, 21.44, 21.22, 21.03, 20.9, 20.84],
  [22.25, 22.53, 22.81, 23.04, 23.21, 23.27, 23.21, 23.01, 22.66, 22.18, 21.62, 21.04, 20.51, 20.09, 19.83, 19.76, 19.88, 20.16, 20.55, 20.99, 21.41, 21.76, 22.0, 22.11, 22.1, 21.99, 21.81, 21.59, 21.37, 21.18, 21.05, 20.99],
  [22.4, 22.68, 22.96, 23.19, 23.36, 23.42, 23.36, 23.16, 22.81, 22.33, 21.77, 21.19, 20.66, 20.24, 19.98, 19.91, 20.03, 20.31, 20.7, 21.14, 21.56, 21.91, 22.15, 22.26, 22.25, 22.14, 21.96, 21.74, 21.52, 21.33, 21.2, 21.14],
  [22.55, 22.83, 23.11, 23.34, 23.51, 23.57, 23.51, 23.31, 22.96, 22.48, 21.92, 21.34, 20.81, 20.39, 20.13, 20.06, 20.18, 20.46, 20.85, 21.29, 21.71, 22.06, 22.3, 22.41, 22.4, 22.29, 22.11, 21.89, 21.67, 21.48, 21.35, 21.29],
  [22.7, 22.98, 23.26, 23.49, 23.66, 23.72, 23.66, 23.46, 23.11, 22.63, 22.07, 21.49, 20.96, 20.54, 20.28, 20.21, 20.33, 20.61, 21.0, 21.44, 21.86, 22.21, 22.45, 22.56, 22.55, 22.44, 22.26, 22.04, 21.82, 21.63, 21.5, 21.44],
  [22.85, 23.13, 23.41, 23.64, 23.81, 23.87, 23.81, 23.61, 23.26, 22.78, 22.22, 21.64, 21.11, 20.69, 20.43, 20.36, 20.48, 20.76, 21.15, 21.59, 22.01, 22.36, 22.6, 22.71, 22.7, 22.59, 22.41, 22.19, 21.97, 21.78, 21.65, 21.59],
  [23.0, 23.28, 23.56, 23.79, 23.96, 24.02, 23.96, 23.76, 23.41, 22.93, 22.37, 21.79, 21.26, 20.84, 20.58, 20.51, 20.63, 20.91, 21.3, 21.74, 22.16, 22.51, 22.75, 22.86, 22.85, 22.74, 22.56, 22.34, 22.12, 21.93, 21.8, 21.74],
  [23.15, 23.43, 23.71, 23.94, 24.11, 24.17, 24.11, 23.91, 23.56, 23.08, 22.52, 21.94, 21.41, 20.99, 20.73, 20.66, 20.78, 21.06, 21.45, 21.89, 22.31, 22.66, 22.9, 23.01, 23.0, 22.89, 22.71, 22.49, 22.27, 22.08, 21.95, 21.89],
  [23.31, 22.67, 22.01, 21.57, 21.32, 21.06, 21.23, 21.65, 22.21, 22.68, 23.04, 23.41, 23.79, 23.93, 24.0, 24.03, 24.34, 24.65, 24.68, 24.68, 24.38, 24.23, 24.23, 24.23, 24.19, 24.28, 24.63, 24.87, 24.95, 24.99, 24.84, 24.74]
];

// 机器人统计数据
const robotStats = ref({
  total: 12,
  change: 2,
  lowBatteryCount: 3,
  offline: 1,
  offlineChange: '-1'
});

// 预警统计数据
const alertStats = ref({
  current: 3,
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

// 选中的机器人 ID 列表
const selectedRobots = ref(['R001', 'R002', 'R003']);

// 计算属性：只显示选中的机器人
const visibleRobots = computed(() => {
  return robots.value.filter(robot => selectedRobots.value.includes(robot.id));
});

// 监听选中状态变化
watch(selectedRobots, (newSelected) => {
  robots.value.forEach(robot => {
    robot.visible = newSelected.includes(robot.id);
  });
}, { deep: true });

// 热力点数据
const heatPoints = ref([
  { x: 100, y: 80, temp: 20, radius: 30 },
  { x: 200, y: 150, temp: 32, radius: 25 },
  { x: 350, y: 180, temp: 42, radius: 40 },
  { x: 450, y: 90, temp: 28, radius: 20 },
  { x: 280, y: 250, temp: 48, radius: 45 }
]);

// AMG8833 热成像数据相关
const showThermalOverlay = ref(true);
const gridSize = ref(32);
let thermalGrid = ref(heatData32x32);

// 视频流数据
const videoStreamActive = ref(true);
const currentVideoFrame = ref('');
const showVideoStream = ref(false);
const showDropdown = ref(false);

// 温度折线图相关
const temperatureChart = ref(null);
let chartInstance = null;
const temperatureHistory = ref([]);

// 切换视图
const toggleView = (view) => {
  showVideoStream.value = (view === 'video');
};

// 切换下拉菜单显示
const toggleDropdown = () => {
  showDropdown.value = !showDropdown.value;
};

// 初始化温度折线图
const initTemperatureChart = () => {
  if (!temperatureChart.value) return;
  
  chartInstance = echarts.init(temperatureChart.value);
  
  // 初始化历史数据
  const now = new Date();
  for (let i = 23; i >= 0; i--) {
    const time = new Date(now.getTime() - i * 3600000);
    temperatureHistory.value.push({
      time: time.toLocaleTimeString('zh-CN', { hour: '2-digit', minute: '2-digit' }),
      temp: 20 + Math.random() * 5
    });
  }
  
  const option = {
    tooltip: {
      trigger: 'axis',
      formatter: '{b}<br/>温度：{c}°C'
    },
    grid: {
      left: '3%',
      right: '4%',
      bottom: '3%',
      top: '15%',
      containLabel: true
    },
    xAxis: {
      type: 'category',
      data: temperatureHistory.value.map(item => item.time),
      axisLabel: {
        fontSize: 10,
        color: '#7f8c8d'
      },
      axisLine: {
        lineStyle: {
          color: '#e0e0e0'
        }
      }
    },
    yAxis: {
      type: 'value',
      name: '温度 (°C)',
      nameTextStyle: {
        fontSize: 10,
        color: '#7f8c8d'
      },
      axisLabel: {
        fontSize: 10,
        color: '#7f8c8d',
        formatter: '{value}°C'
      },
      splitLine: {
        lineStyle: {
          color: '#f0f0f0'
        }
      }
    },
    series: [{
      name: '温度',
      type: 'line',
      data: temperatureHistory.value.map(item => item.temp),
      smooth: true,
      symbol: 'circle',
      symbolSize: 6,
      itemStyle: {
        color: '#ff6b6b'
      },
      lineStyle: {
        width: 2,
        color: '#ff6b6b'
      },
      areaStyle: {
        color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
          { offset: 0, color: 'rgba(255, 107, 107, 0.3)' },
          { offset: 1, color: 'rgba(255, 107, 107, 0.05)' }
        ])
      }
    }]
  };
  
  chartInstance.setOption(option);
  
  // 响应式调整
  window.addEventListener('resize', handleChartResize);
};

// 图表大小调整
const handleChartResize = () => {
  if (chartInstance) {
    chartInstance.resize();
  }
};

// 更新温度数据
const updateTemperatureData = (newTemp) => {
  const now = new Date();
  const timeStr = now.toLocaleTimeString('zh-CN', { hour: '2-digit', minute: '2-digit' });
  
  temperatureHistory.value.push({
    time: timeStr,
    temp: newTemp
  });
  
  // 保持最近 24 小时数据
  if (temperatureHistory.value.length > 24) {
    temperatureHistory.value.shift();
  }
  
  // 更新图表
  if (chartInstance) {
    chartInstance.setOption({
      xAxis: {
        data: temperatureHistory.value.map(item => item.time)
      },
      series: [{
        data: temperatureHistory.value.map(item => item.temp)
      }]
    });
  }
};

const getHeatmapColor = (temperature) => {
  if (temperature < 20) return 'rgba(33, 150, 243, 0.6)';
  if (temperature <= 30) return 'rgba(33, 150, 243, 0.8)';
  if (temperature <= 35) return 'rgba(255, 193, 7, 0.6)';
  if (temperature <= 40) return 'rgba(255, 152, 0, 0.7)';
  return 'rgba(244, 67, 54, 0.6)';
};

const getThermalColor = (temp) => {
  const minTemp = 7.8;
  const maxTemp = 24.68;
  const ratio = Math.min(Math.max((temp - minTemp) / (maxTemp - minTemp), 0), 1);

  let r, g, b;
  if (ratio < 0.5) {
    const t = ratio * 2;
    r = Math.floor(255 * t);
    g = Math.floor(255 * Math.min(t * 2, 1));
    b = Math.floor(255 * (1 - t));
  } else {
    const t = (ratio - 0.5) * 2;
    r = 255;
    g = Math.floor(255 * (1 - t));
    b = 0;
  }

  return `rgb(${r}, ${g}, ${b})`;
};

const getOpacityByTemp = (temp) => {
  const minTemp = 7.8;
  const maxTemp = 24.68;
  const ratio = Math.min(Math.max((temp - minTemp) / (maxTemp - minTemp), 0), 1);
  return 0.3 + ratio * 0.7;
};

const toggleGridSize = () => {
  if (gridSize.value === 32) {
    gridSize.value = 8;
    thermalGrid.value = original_8x8;
  } else {
    gridSize.value = 32;
    thermalGrid.value = heatData32x32;
  }
};

// SSE 连接
let eventSource = null;

const initSSEConnection = () => {
  if (eventSource) {
    eventSource.close();
  }

  eventSource = new EventSource('/api/sse/stream');

  eventSource.onmessage = (event) => {
    try {
      const data = JSON.parse(event.data);
      updateDashboardData(data);
    } catch (error) {
      console.error('解析 SSE 数据失败:', error);
    }
  };

  eventSource.onerror = (error) => {
    console.error('SSE 连接错误:', error);
    eventSource.close();
    setTimeout(initSSEConnection, 5000);
  };

  eventSource.onopen = () => {
    console.log('SSE 连接已建立 - /api/sse/stream');
  };
};

const updateDashboardData = (data) => {
  if (data.robotStats) {
    robotStats.value = {
      total: data.robotStats.total || robotStats.value.total,
      change: data.robotStats.change || robotStats.value.change,
      lowBatteryCount: data.robotStats.lowBatteryCount || 0,
      offline: data.robotStats.offline || 0,
      offlineChange: data.robotStats.offlineChange || robotStats.value.offlineChange
    };
  }

  if (data.alertStats) {
    alertStats.value = {
      current: data.alertStats.current || 0,
      week: data.alertStats.week || alertStats.value.week
    };
  }

  if (data.environment) {
    environmentData.value = {
      temperature: data.environment.temperature || environmentData.value.temperature,
      humidity: data.environment.humidity || environmentData.value.humidity,
      gas: data.environment.gas || environmentData.value.gas,
      pm25: data.environment.pm25 || environmentData.value.pm25
    };
    
    // 更新温度折线图
    updateTemperatureData(environmentData.value.temperature);
  }

  if (data.robots && Array.isArray(data.robots)) {
    robots.value = data.robots.map(robot => ({
      id: robot.id,
      x: robot.x || Math.random() * 400 + 50,
      y: robot.y || Math.random() * 300 + 50,
      battery: robot.battery || Math.floor(Math.random() * 100),
      moving: robot.moving || Math.random() > 0.5
    }));
  }

  if (data.heatPoints && Array.isArray(data.heatPoints)) {
    heatPoints.value = data.heatPoints.map(point => ({
      x: point.x || Math.random() * 500,
      y: point.y || Math.random() * 350,
      temp: point.temp || Math.random() * 30 + 10,
      radius: point.radius || Math.random() * 30 + 10
    }));
  }

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
  initSSEConnection();
  initTemperatureChart();
  
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
  if (eventSource) {
    eventSource.close();
    eventSource = null;
  }
  
  if (chartInstance) {
    chartInstance.dispose();
    chartInstance = null;
  }
  
  window.removeEventListener('resize', handleChartResize);
});
</script>

<style scoped>
/* 全局样式 */
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

.view-toggle-panel {
  margin-top: 5px;
  display: flex;
  gap: 5px;
  justify-content: center;
}

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

/* 温度折线图容器样式 */
.chart-container {
  flex: 1;
  min-height: 0;
  width: 100%;
  height: 100%;
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
  
  .environment-grid {
    grid-template-columns: repeat(2, 1fr);
  }
  
  .stats-grid {
    grid-template-columns: 1fr;
  }
  
  .map-container {
    height: 200px;
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