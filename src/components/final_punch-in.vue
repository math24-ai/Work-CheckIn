<!--suppress JSUnresolvedReference -->
<template>
  <div class="app-container">
    <!-- 时间显示 -->
    <timeShow />

    <!-- 打卡状态 -->
    <span v-if="clockCount === 0" class="tag tag-red">请上班打卡</span>
    <span v-if="clockCount === 1" class="tag tag-green">请下班打卡</span>
    <span v-if="clockCount === 2" class="tag tag-blue">今天的打卡完成</span>

    <!-- 简单功能 -->
    <div class="storageOperate">
      <button @click="addRandomRecord" class="btn-primary">添加随机记录</button>
      <button @click="resetLocalStorage" class="btn-primary">重置本地存储</button>
    </div>
    <div class="clock-actions">
      <button class="btn-primary" @click="handleClockInOut" :disabled="clockCount >= 2">打卡</button>
    </div>

    <!-- 打卡记录 -->
    <transition-group name="fade" tag="table" class="table">
      <thead>
      <tr>
        <th>序号</th>
        <th>打卡日期</th>
        <th>上班打卡时间</th>
        <th>下班打卡时间</th>
        <th>打卡次数</th>
        <th>操作</th>
      </tr>
      </thead>
      <tbody>
      <tr v-for="record in getPageRecords(clockRecords)" :key="record.index" class="fade-item">
        <td>{{ record.index }}</td>
        <td>{{ record.clockDate }}</td>
        <td>{{ record.workStart }}</td>
        <td>{{ record.workEnd }}</td>
        <td>{{ record.clockCount }}</td>
        <td><button class="delete-btn" @click="handleDelete(record.index)">删除</button></td>
      </tr>
      </tbody>
    </transition-group>

    <!-- 分页按钮 -->
    <div v-if="clockRecords.length > itemsPerPage" class="pagination">
      <button @click="handlePageChange('prev')" :disabled="currentPage === 1">上一页</button>
      <span>第 {{ currentPage }} 页</span>
      <button @click="handlePageChange('next')" :disabled="clockRecords.length <= currentPage * itemsPerPage">下一页</button>
    </div>

    <!-- 垃圾桶表格 -->
    <div v-if="trashRecords.length > 0" class="trashCan">
      <h3>垃圾桶</h3>
      <transition-group name="fade" tag="table" class="table">
        <thead>
        <tr>
          <th>序号</th>
          <th>打卡日期</th>
          <th>操作</th>
        </tr>
        </thead>
        <tbody>
        <tr v-for="(record, idx) in trashRecords" :key="record.index">
          <td>{{ idx + 1 }}</td>
          <td>{{ record.clockDate }}</td>
          <td>
            <button class="restore-btn" @click="handleRestore(record.index)">恢复</button>
            <button class="delete-btn" @click="handlePermanentlyDelete(record.index)">彻底删除</button>
          </td>
        </tr>
        </tbody>
      </transition-group>
    </div>

    <!-- ECharts 图表 -->
    <div ref="chart" class="chart-container"></div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import * as echarts from 'echarts';
import timeShow from './timeShow.vue';

// 默认打卡记录数据
const defaultClockRecords = [
  { index: 1, clockDate: '2024-11-27', workStart: '07:00', workEnd: '18:00', clockCount: 2 },
  { index: 2, clockDate: '2024-11-26', workStart: '06:45', workEnd: '17:45', clockCount: 2 },
  { index: 3, clockDate: '2024-11-25', workStart: '08:12', workEnd: '17:51', clockCount: 2 },
  { index: 4, clockDate: '2024-11-24', workStart: '09:31', workEnd: '18:16', clockCount: 2 },
  { index: 5, clockDate: '2024-11-22', workStart: '05:31', workEnd: '20:16', clockCount: 2 },
  { index: 6, clockDate: '2024-11-18', workStart: '09:00', workEnd: '16:16', clockCount: 2 },
  { index: 7, clockDate: '2024-11-16', workStart: '06:31', workEnd: '20:24', clockCount: 2 },
  { index: 8, clockDate: '2024-11-15', workStart: '08:31', workEnd: '19:45', clockCount: 2 },
];

// 初始化数据
const clockRecords = ref([]);
const trashRecords = ref([]);
let clockCount = ref(0);
let currentDate = `${new Date().getFullYear()}-${(new Date().getMonth() + 1).toString().padStart(2, '0')}-${new Date().getDate().toString().padStart(2, '0')}`;
let myChart = null;
const itemsPerPage = 7;
const currentPage = ref(1);

// 加载本地存储数据
const loadFromLocalStorage = () => {
  const savedClockRecords = localStorage.getItem('clockRecords');
  const savedTrashRecords = localStorage.getItem('trashRecords');
  const savedClockCount = localStorage.getItem('clockCount');

  if (savedClockRecords) clockRecords.value = JSON.parse(savedClockRecords);
  if (savedTrashRecords) trashRecords.value = JSON.parse(savedTrashRecords);
  if (savedClockCount) clockCount.value = JSON.parse(savedClockCount);
  else {
    clockRecords.value = [...defaultClockRecords];
    saveToLocalStorage(); // 保存初始化数据
  }
};

// 保存数据到本地存储
const saveToLocalStorage = () => {
  localStorage.setItem('clockRecords', JSON.stringify(clockRecords.value));
  localStorage.setItem('trashRecords', JSON.stringify(trashRecords.value));
  localStorage.setItem('clockCount', JSON.stringify(clockCount.value));
};

// 重置本地存储
const resetLocalStorage = () => {
  // 清空本地存储
  localStorage.removeItem('clockRecords');
  localStorage.removeItem('trashRecords');
  localStorage.removeItem('clockCount');

  // 使用默认数据重新初始化
  clockRecords.value = [...defaultClockRecords]; // 重新赋值为默认数据
  trashRecords.value = []; // 清空垃圾桶
  clockCount.value = 0; // 重置打卡状态

  saveToLocalStorage(); // 保存初始化数据到本地存储
};

// 计算当前页的起始索引和结束索引
const getPageRecords = (records) => {
  const startIndex = (currentPage.value - 1) * itemsPerPage;
  const endIndex = startIndex + itemsPerPage;
  return records.slice(startIndex, endIndex); // 返回当前页的数据
};

// 翻页操作
const handlePageChange = (direction) => {
  if (direction === 'next') {
    currentPage.value++;
  } else if (direction === 'prev' && currentPage.value > 1) {
    currentPage.value--;
  }
};

// 更新打卡记录序号
const updateIndex = () => {
  clockRecords.value.forEach((record, index) => {
    record.index = index + 1;
  });
};

// 排序函数
const sortRecordsByDate = () => {
  setTimeout(() => {
    clockRecords.value.sort((a, b) => new Date(b.clockDate) - new Date(a.clockDate));
    updateIndex();
    updateChartData();
    saveToLocalStorage();
  }, 500); // .5秒的延迟
};


// 转换时间为分钟
const timeToMinutes = (time) => {
  const [hours, minutes] = time.split(':').map(Number);
  return hours * 60 + minutes;
};

// 添加随机记录
const getRandomTime = (minHour, maxHour) => {
  const randomHour = Math.floor(Math.random() * (maxHour - minHour + 1)) + minHour;
  const randomMinute = Math.floor(Math.random() * 60);
  return `${randomHour.toString().padStart(2, '0')}:${randomMinute.toString().padStart(2, '0')}`;
};

const generateRandomDate = () => {
  const currentDate = new Date();
  const randomOffset = Math.floor(Math.random() * 30); // 随机生成过去30天内的日期
  currentDate.setDate(currentDate.getDate() - randomOffset);
  return `${currentDate.getFullYear()}-${(currentDate.getMonth() + 1).toString().padStart(2, '0')}-${currentDate.getDate().toString().padStart(2, '0')}`;
};

// 添加随机记录的逻辑
const addRandomRecord = () => {
  let randomDate = generateRandomDate();

  // 确保生成的日期没有重复
  while (clockRecords.value.some(record => record.clockDate === randomDate)) {
    randomDate = generateRandomDate();
  }

  const workStart = getRandomTime(6, 8);  // 上班时间在 6:00-8:00 之间
  const workEnd = getRandomTime(16, 22);  // 下班时间在 16:00-22:00 之间

  clockRecords.value.push({
    index: clockRecords.value.length + 1,
    clockDate: randomDate,
    workStart,
    workEnd,
    clockCount: 2,
  });

  sortRecordsByDate();
  updateChartData();
  saveToLocalStorage();
};

// 获取当天的打卡记录
const getTodayRecord = () => {
  return clockRecords.value.find(record => record.clockDate === currentDate);
};

// 打卡操作
const handleClockInOut = () => {
  const todayRecord = getTodayRecord(); // 获取当天的打卡记录

  if (todayRecord) {
    if (todayRecord.clockCount === 2) {
      alert('今天已经打卡两次，无法再次打卡！');
      return;
    } else if (todayRecord.clockCount === 1) {
      todayRecord.workEnd = `${new Date().getHours().toString().padStart(2, '0')}:${new Date().getMinutes().toString().padStart(2, '0')}`;
      todayRecord.clockCount = 2;
      clockCount.value = 2; // 下班打卡后，设置 clockCount 为 2
    }
  } else {
    // 如果今天没有打卡记录，则创建一条上班记录
    const currentTime = `${new Date().getHours().toString().padStart(2, '0')}:${new Date().getMinutes().toString().padStart(2, '0')}`;
    clockRecords.value.push({
      index: clockRecords.value.length + 1,
      clockDate: currentDate,
      workStart: currentTime,
      workEnd: '-',
      clockCount: 1,
    });
    clockCount.value = 1; // 上班打卡后，设置 clockCount 为 1
  }

  sortRecordsByDate();
  updateChartData();
  saveToLocalStorage();
};

// 删除操作
let deleteLock = false; // 删除锁，防止多次快速点击
const handleDelete = (index) => {
  if (deleteLock) return; // 如果锁定，则返回

  deleteLock = true; // 设置锁定状态，开始删除操作
  const recordToDelete = clockRecords.value.find(record => record.index === index);

  if (recordToDelete) {
    if (recordToDelete.clockDate === currentDate) {
      clockCount.value = 0; // 设置为 0 显示红色提示
    }
    trashRecords.value.push(recordToDelete); // 将记录移至垃圾桶
    clockRecords.value = clockRecords.value.filter(record => record.index !== index); // 从原记录中删除
    sortRecordsByDate();
    updateChartData();  // 更新图表
    saveToLocalStorage();

    // 设置2秒的缓冲时间，结束后解除锁定
    setTimeout(() => {
      deleteLock = false; // 解除锁定状态
    }, 2000);
  }
};

// 恢复操作
let restoreLock = false;
const handleRestore = (index) => {
  if (restoreLock) return;

  restoreLock = true;
  const recordToRestore = trashRecords.value.find(record => record.index === index);
  if (recordToRestore) {
    const existingRecord = clockRecords.value.find(record => record.clockDate === recordToRestore.clockDate);

    if (existingRecord) {
      const confirmReplace = confirm('今天已有打卡记录，是否替换？');
      if (confirmReplace) {
        clockRecords.value = clockRecords.value.filter(record => record.index !== existingRecord.index); // 删除旧记录
        clockRecords.value.push(recordToRestore); // 恢复记录
        clockCount.value = recordToRestore.clockCount;
        trashRecords.value = trashRecords.value.filter(record => record.index !== index);
        sortRecordsByDate();
        updateChartData();
        saveToLocalStorage();
      }
    } else {
      clockRecords.value.push(recordToRestore); // 如果没有重复记录，直接恢复
      clockCount.value = recordToRestore.clockCount;
      trashRecords.value = trashRecords.value.filter(record => record.index !== index);
      sortRecordsByDate();
      updateChartData();
      saveToLocalStorage();
    }
  }

  setTimeout(() => {
    restoreLock = false;
  }, 2000);
};

// 彻底删除操作
let deletePermanentLock = false;
const handlePermanentlyDelete = (index) => {
  if (deletePermanentLock) return;

  deletePermanentLock = true;
  trashRecords.value = trashRecords.value.filter(record => record.index !== index);
  saveToLocalStorage();

  setTimeout(() => {
    deletePermanentLock = false;
  }, 2000);
};


// 更新图表数据
const updateChartData = () => {
  const chartData = prepareChartData();
  if (myChart) {
    myChart.setOption({
      title: {
        text: '可视化：近7天内数据',
        left: 'center',
        textStyle: {
          fontSize: 16,
          fontWeight: 'bold',
          color: '#333',
        },
      },
      grid: {
        top: '20%',
      },
      legend: {
        data: ['上班时间', '下班时间'],
        left: 'center',
        top: '30px',
      },
      xAxis: {
        data: chartData.dates,
        axisLabel: { interval: 0 },
        name: '日期',
        axisLine: {
          lineStyle: {
            color: '#333',
            width: 2,
            type: 'solid',
          },
        },
        splitLine: {
          show: true,
          lineStyle: {
            color: '#ddd',
            type: 'dashed',
          },
        },
      },
      yAxis: {
        type: 'value',
        name: '打卡时间',
        max: 1440,
        axisLabel: {
          formatter: (value) => `${Math.floor(value / 60)}小时`, // 显示小时单位
        },
        axisLine: {
          show: true,
          lineStyle: {
            color: '#333',
            width: 2,
            type: 'solid',
          },
        },
        splitLine: {
          show: true,
          lineStyle: {
            color: '#ddd',
            type: 'dashed',
          },
        },
      },
      series: [
        { name: '上班时间', type: 'line', data: chartData.workStartTimes },
        { name: '下班时间', type: 'line', data: chartData.workEndTimes },
      ],
    });
  }
};

// 准备图表数据
const prepareChartData = () => {
  const dates = [];
  const workStartTimes = [];
  const workEndTimes = [];
  // 只取前 7 条记录
  const recordsToShow = clockRecords.value.slice(0, 7);

  recordsToShow.forEach(record => {
    dates.push(record.clockDate);
    workStartTimes.push(timeToMinutes(record.workStart));
    workEndTimes.push(record.workEnd === '-' ? null : timeToMinutes(record.workEnd));
  });

  return { dates, workStartTimes, workEndTimes };
};


// 绘制图表
const drawChart = () => {
  const chartData = prepareChartData();
  myChart = echarts.init(document.querySelector('.chart-container'));
  myChart.setOption({
    legend: { data: ['上班时间', '下班时间'], left: 'center' },
    tooltip: { trigger: 'axis', formatter: (params) => {
        const date = params[0].name;
        const workStart = params[0].value;
        const workEnd = params[1].value;
        return `${date}<br />上班：${Math.floor(workStart / 60)}:${String(workStart % 60).padStart(2, '0')}<br />下班：${workEnd ? `${Math.floor(workEnd / 60)}:${String(workEnd % 60).padStart(2, '0')}` : '未打卡'}`;
      } },
    xAxis: { data: chartData.dates },
    yAxis: { type: 'value', axisLabel: { formatter: (value) => `${Math.floor(value / 60)}小时` } },
    series: [
      { name: '上班时间', type: 'line', data: chartData.workStartTimes },
      { name: '下班时间', type: 'line', data: chartData.workEndTimes },
    ],
  });

  updateChartData();
};

onMounted(() => {
  if (!sessionStorage.getItem('refreshed')) {
    sessionStorage.setItem('refreshed', 'true');
    window.location.reload();
  } else {
    loadFromLocalStorage();
    const currentTime = new Date();
    const currentDate = `${currentTime.getFullYear()}-${(currentTime.getMonth() + 1).toString().padStart(2, '0')}-${currentTime.getDate().toString().padStart(2, '0')}`;

    // 如果当前时间已经过了6:00点，且当天没有打卡记录，只更新状态为“请上班打卡”
    if (currentTime.getHours() >= 6) {
      const todayRecord = clockRecords.value.find(record => record.clockDate === currentDate);
      if (!todayRecord) {
        clockCount.value = 0;
      }
    }

    drawChart();
  }
});
</script>

<style scoped>
.app-container {
  padding: 30px;
  max-width: 1200px;
  margin: 0 auto;
  background: #ffffff;
  border-radius: 12px;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
  position: relative;
  user-select: none;
}

h2 {
  font-size: 28px;
  font-weight: 600;
  color: #333;
  margin-bottom: 24px;
}

.storageOperate {
  --width: 30px;
  --height: 158px;
  position: absolute;
  right: var(--width);
  top: var(--height);

  & > button {
    border-radius: 12px;
    font-weight: 600;
  }
}

.clock-actions {
  display: flex;
  justify-content: center;
  margin-bottom: 20px;
}

.tag {
  display: inline-block;
  padding: 6px 20px;
  border-radius: 12px;
  font-weight: 600;
}

.tag-red {
  background-color: #e74c3c;
  color: white;
}

.tag-green {
  background-color: #2ecc71;
  color: white;
}

.tag-blue {
  background-color: rgb(135,206,250);
  color: white;
}

.btn-primary {
  padding: 8px 16px;
  background-color: #007bff;
  border: none;
  border-radius: 4px;
  color: white;
  cursor: pointer;
  transition: background-color 0.3s;
}

.btn-primary:hover {
  background-color: #0056b3;
}

.table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 30px;
}

.table th, .table td {
  padding: 12px 20px;
  border: 1px solid #e4e7ed;
  text-align: center;
}

.table th {
  background-color: #f0f4f8;
  font-weight: 600;
}

.delete-btn, .restore-btn, .permanent-delete-btn {
  padding: 6px 12px;
  background-color: #ff4d4f;
  border: none;
  color: white;
  border-radius: 4px;
  cursor: pointer;
}

.pagination {
  width: 100%;
  display: flex;
  justify-content: center;
  margin-top: 30px;

  & > span {
    margin: 0 5px 0 5px;
  }
}

.restore-btn {
  background-color: #2ecc71;
}

.permanent-delete-btn {
  background-color: #f39c12;
}

.delete-btn:hover, .restore-btn:hover, .permanent-delete-btn:hover {
  opacity: 0.8;
}

.chart-container {
  height: 400px;
  margin-top: 30px;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
  background-color: #fff;
  padding: 30px;
}

.trash-container {
  display: flex;
  flex-direction: column;
  margin-top: 20px;
}

.trash-item {
  display: flex;
  justify-content: space-between;
  padding: 10px;
  background-color: #f9f9f9;
  border-radius: 4px;
  margin-bottom: 10px;
}

.trash-item button {
  margin-left: 10px;
}

.trash-item .permanent-delete-btn {
  background-color: #e74c3c;
}
</style>
