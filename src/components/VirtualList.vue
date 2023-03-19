<template>
  <div
    ref="listWrapper"
    :style="{ height: containerHeight }"
    class="infinite-list-container"
    @scroll="(e) => scrollEvent(e)"
  >
    <div ref="phantom" class="infinite-list-phantom"></div>
    <div ref="content" class="infinite-list">
      <div
        class="infinite-list-item"
        ref="items"
        :id="item._index"
        :key="item._index"
        v-for="item in visibleData"
      >
        <slot ref="slot" :item="item.item"></slot>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onUpdated, watch, onMounted, computed, nextTick } from 'vue'
export interface PropsType {
  listData?: Array<Record<string, any>>
  estimatedItemSize: number
  bufferScale?: number
  containerHeight?: string
  useLoading?: boolean
}

interface PositionType {
  index: number
  height: number
  top: number
  bottom: number
}

const props = withDefaults(defineProps<PropsType>(), {
  listData: () => [], //所有列表数据
  estimatedItemSize: 100, //预估高度
  bufferScale: 3, //缓冲区比例
  containerHeight: '100%', // 容器高度 100px
  useLoading: false // 是否使用 loading 动画
})

const emit = defineEmits<{
  (e: 'reachBottom'): void
}>()

let positions: PositionType[]
let screenHeight = 0
let visibleCount = 0
const start = ref(0)
const end = ref(0)
const items = ref<HTMLElement[] | null>(null)
const phantom = ref<HTMLElement | null>(null)
const listWrapper = ref<HTMLElement | null>(null)
const content = ref<HTMLElement | null>(null)
const aboveCount = ref(0)
const belowCount = ref(0)

const _listData = computed(() =>
  props.listData.map((item, index) => {
    return {
      _index: `_${index}`,
      item
    }
  })
)

const visibleData = computed(() => {
  let newStart = start.value - aboveCount.value
  let newEnd = end.value + belowCount.value
  return _listData.value.slice(newStart, newEnd)
})

function createPositions(start: number, end: number) {
  let list: PositionType[] = []
  for (let index = start; index < end; index++) {
    list.push({
      index,
      height: props.estimatedItemSize,
      top: index * props.estimatedItemSize,
      bottom: (index + 1) * props.estimatedItemSize
    })
  }
  return list
}

function binarySearch(list: PositionType[], val: number) {
  let startVal = 0
  let endVal = list.length - 1
  let tempIndex = null

  while (startVal <= endVal) {
    let midIndex = ~~((startVal + endVal) / 2)
    let midValue = list[midIndex].bottom
    if (midValue === val) {
      return midIndex + 1
    } else if (midValue < val) {
      startVal = midIndex + 1
    } else if (midValue > val) {
      if (tempIndex === null || tempIndex > midIndex) {
        tempIndex = midIndex
      }
      endVal = endVal - 1
    }
  }
  return Number(tempIndex)
}

function updateItemsSize() {
  let nodes = items.value
  nodes!.forEach((node) => {
    let rect = node.getBoundingClientRect()
    let height = rect.height
    let index = +node.id.slice(1)
    let oldHeight = positions[index].height
    let dValue = oldHeight - height
    //存在差值
    if (dValue) {
      positions[index].bottom = positions[index].bottom - dValue
      positions[index].height = height
      for (let k = index + 1; k < positions.length; k++) {
        positions[k].top = positions[k - 1].bottom
        positions[k].bottom = positions[k].bottom - dValue
      }
    }
  })
}

function setStartOffset() {
  let startOffset
  if (start.value >= 1) {
    let size =
      positions[start.value].top -
      (positions[start.value - aboveCount.value]
        ? positions[start.value - aboveCount.value].top
        : 0)
    startOffset = positions[start.value - 1].bottom - size
  } else {
    startOffset = 0
  }
  content.value!.style.transform = `translateY(${startOffset}px)`
}

function scrollEvent(e: Event) {
  //当前滚动位置
  let innerScrollTop = (e.target as HTMLElement).scrollTop
  let innerScrollHeight = (e.target as HTMLElement).scrollHeight
  //此时的开始索引
  start.value = binarySearch(positions, innerScrollTop)
  //此时的结束索引
  end.value = start.value + visibleCount
  //此时的偏移量
  setStartOffset()
  if (screenHeight + innerScrollTop >= innerScrollHeight - 1) {
    console.log('reach bottom')
    emit('reachBottom')
  }
}

positions = createPositions(0, props.listData.length)

onMounted(() => {
  start.value = 0
  end.value = start.value + visibleCount
  screenHeight = listWrapper.value!.clientHeight
  visibleCount = screenHeight / props.estimatedItemSize
  aboveCount.value = Math.min(start.value, props.bufferScale * visibleCount)
  belowCount.value = Math.min(props.listData.length - end.value, props.bufferScale * visibleCount)
})

onUpdated(() => {
  nextTick(function () {
    if (!items.value || !items.value?.length) {
      return
    }
    //获取真实元素大小，修改对应的尺寸缓存
    updateItemsSize()
    //更新列表总高度
    let height = positions[positions.length - 1].bottom
    phantom.value!.style.height = height + 'px'
    //更新真实偏移量
    setStartOffset()
  })
})

watch(
  () => props.listData.length,
  (newLen, oldLen) => {
    positions = positions.concat(createPositions(oldLen, newLen))
  }
)
</script>

<style scoped>
.infinite-list-container {
  overflow: auto;
  position: relative;
  -webkit-overflow-scrolling: touch;
}

.infinite-list-phantom {
  position: absolute;
  left: 0;
  top: 0;
  right: 0;
  z-index: -1;
}

.infinite-list {
  left: 0;
  right: 0;
  top: 0;
  position: absolute;
}
</style>
