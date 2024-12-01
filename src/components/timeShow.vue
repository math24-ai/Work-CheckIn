<template>
  <div id="clock">
    <p class="date">{{ date }}</p>
    <p class="time">{{ time }}</p>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

const time = ref('');
const date = ref('');
const week = ['周日', '周一', '周二', '周三', '周四', '周五', '周六'];

const updateTime = () => {
  const cd = new Date();
  time.value = zeroPadding(cd.getHours(), 2) + ':' + zeroPadding(cd.getMinutes(), 2) + ':' + zeroPadding(cd.getSeconds(), 2);
  date.value = zeroPadding(cd.getFullYear(), 4) + '-' + zeroPadding(cd.getMonth() + 1, 2) + '-' + zeroPadding(cd.getDate(), 2) + ' ' + week[cd.getDay()];
};

const zeroPadding = (num, digit) => {
  let zero = '';
  for (let i = 0; i < digit; i++) {
    zero += '0';
  }
  return (zero + num).slice(-digit);
};

onMounted(() => {
  const timerID = setInterval(updateTime, 1000);
  updateTime();

  onUnmounted(() => {
    clearInterval(timerID);
  });
});
</script>

<style scoped>
p {
  margin: 0;
  padding: 0;
}

#clock {
  font-family: 'Share Tech Mono', monospace;
  color: rgba(0,0,0,0.8);
  text-align: center;
  position: relative;
  left: 50%;
  z-index: 99;
  transform: translate(-50%, 10px);
  text-shadow: 0 0 20px rgba(10, 175, 230, 1), 0 0 20px rgba(10, 175, 230, 0);
}

.time {
  letter-spacing: 0.05em;
  font-size: 80px;
  padding: 5px 0;
}

.date {
  letter-spacing: 0.1em;
  font-size: 24px;
}

.text {
  letter-spacing: 0.1em;
  font-size: 12px;
  padding: 20px 0 0;
}
</style>
