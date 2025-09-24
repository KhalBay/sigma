<script setup lang="ts">
import { computed, ref } from "vue";
import firstTab from './components/FirstTask.vue'
import secondTab from './components/SecondTask.vue'

const activeNum = ref<number>(1)

const activeTab = computed(() => {
  switch (activeNum.value) {
    case 1:
      return firstTab
    case 2:
      return secondTab
    default:
      return firstTab
  }
})

const setActiveNum = (num: number) => {
  activeNum.value = num
}
</script>

<template>
  <div>
    <nav class="flex">
      <div :class="{'active': activeNum === 1}" @click="setActiveNum(1)">Задание 1</div>
      <div :class="{'active': activeNum === 2}" @click="setActiveNum(2)">Задание 2</div>
    </nav>
    <Component :is="activeTab"/>
  </div>
</template>

<style scoped>
.flex {
  display: flex;
  justify-content: space-around;
  margin: 20px;
}

.flex div {
  background: cornflowerblue;
  color: white;
  opacity: .3;
  border-radius: 12px;
  padding: 4px 10px;
  cursor: pointer;
}

.flex div.active {
  opacity: 1;
}
</style>