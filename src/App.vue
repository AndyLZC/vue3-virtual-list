<script setup lang="ts">
import faker from 'faker'
import { ref, onMounted } from 'vue'
import { NButton, NList, NModal, NSpace } from 'naive-ui'
import VirtualList from './components/VirtualList.vue'

interface SentenceType {
  id: number
  value: string
}

const fakerList = ref<SentenceType[]>([])
const showModal1 = ref(false)
const showModal2 = ref(false)
const SCREEN_HEIGHT = 800

// 模拟获取数据
function getFakerList(): Promise<SentenceType[]> {
  return new Promise((resolve) => {
    setTimeout(() => {
      let newFakerList = []
      let len = fakerList.value.length
      for (let id = len; id < len + 10; ++id) {
        newFakerList.push({
          id,
          value: faker.lorem.sentences()
        })
      }
      resolve(newFakerList)
    }, 50)
  })
}

async function handleReachBottom() {
  const list = await getFakerList()
  fakerList.value = [...fakerList.value, ...list]
}

function handleScroll(e: Event) {
  let scrollTop = (e.target as HTMLElement).scrollTop
  let scrollHeight = (e.target as HTMLElement).scrollHeight
  if (SCREEN_HEIGHT + scrollTop > scrollHeight - 30) {
    console.log('reach bottom')
    handleReachBottom()
  }
}

async function handlemModalClose() {
  fakerList.value.length = 0
  handleReachBottom()
}

onMounted(async () => {
  const list = await getFakerList()
  fakerList.value = [...fakerList.value, ...list]
})
</script>

<template>
  <div class="container">
    <n-space vertical>
      <n-button type="warning" @click="showModal1 = true"> 普通列表 </n-button>
      <n-button type="error" @click="showModal2 = true"> 虚拟列表</n-button>
    </n-space>
    <n-modal
      v-model:show="showModal1"
      style="width: 500px"
      preset="dialog"
      :on-close="handlemModalClose"
    >
      <div
        style="height: 800px; overflow: scroll; padding: 20px 10px"
        @scroll="(e) => handleScroll(e)"
      >
        <n-list>
          <n-list-item v-for="item of fakerList" :key="item.id">
            <div class="list-item">
              <span class="id"> Item - {{ item.id }} Data</span>
              <span class="val">{{ item.value }}</span>
            </div>
          </n-list-item>
        </n-list>
      </div>
    </n-modal>
    <n-modal
      v-model:show="showModal2"
      style="width: 500px"
      preset="dialog"
      :on-close="handlemModalClose"
    >
      <div style="height: 800px; padding: 20px 10px">
        <VirtualList
          :listData="fakerList"
          :estimated-item-size="150"
          v-slot="slotProps"
          @reachBottom="handleReachBottom"
        >
          <div class="list-item">
            <span class="id"> Item - {{ slotProps.item.id }} Data</span>
            <span class="val">{{ slotProps.item.value }}</span>
          </div>
        </VirtualList>
      </div>
    </n-modal>
  </div>
</template>

<style scoped>
.container {
  position: absolute;
  left: 0;
  right: 0;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.list-item {
  padding: 20px 0;
  border-bottom: 1px solid #000;
}

.id {
  display: block;
  color: rgba(0, 0, 0, 0, 85);
  font-weight: 500;
  font-size: 14px;
}

.val {
  width: 100%;
  color: rgba(0, 0, 0, 0.5);
  font-size: 16px;
}
</style>
