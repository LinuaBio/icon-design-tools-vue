<template>
  <div class="svg-container">
    <svg ref="svg" @mousemove="handleMouseMove">
      <g>
        <!-- 子元素 -->
        <template v-for="item in subElements" :key="item.id">
          <path :d="item.path" :fill="item.color" @click="handleElementClick(item)" />
        </template>
      </g>
      <!-- S图片 -->
      <image ref="sImage" :x="sImagePosition.x" :y="sImagePosition.y" :width="sImagePosition.width" :height="sImagePosition.height" :href="sImageUrl" />
    </svg>
  </div>
</template>

<script lang="ts">
import { defineComponent} from 'vue'

export default defineComponent({
  data() {
    return {
      subElements: [
        // 设置子元素的初始状态
        { id: 1, path: 'M100 100 L200 100 L200 200 L100 200 Z', color: 'red', sound: 'sound1.mp3', isOverlapping: false },
        { id: 2, path: 'M150 150 L250 150 L250 250 L150 250 Z', color: 'blue', sound: 'sound2.mp3', isOverlapping: false },
        // ...
      ],
      sImageUrl: 's-image.jpg',
      sImagePosition: { x: 0, y: 0, width: 0, height: 0 },
      currentElementIndex: -1
    }
  },
  methods: {
    handleMouseMove(event: MouseEvent) {
      const svgRect = (this.$refs.svg as SVGSVGElement).getBoundingClientRect()
      const x = event.clientX - svgRect.x
      const y = event.clientY - svgRect.y
      const overlappingElements = this.subElements.filter(item => {
        const elementRect = (this.$refs[item.id.toString()] as SVGPathElement).getBoundingClientRect()
        return x >= elementRect.left && x <= elementRect.right && y >= elementRect.top && y <= elementRect.bottom
      })
      const elementIndex = overlappingElements.findIndex(item => item.id === this.currentElementIndex)
      if (elementIndex !== -1) {
        this.currentElementIndex = overlappingElements[elementIndex].id
        overlappingElements.forEach(item => {
          if (item.id > this.currentElementIndex) {
            item.isOverlapping = true
          } else {
            item.isOverlapping = false
          }
        })
        this.setSImagePosition(x, y, svgRect)
      } else {
        this.currentElementIndex = -1
        this.subElements.forEach(item => {
          item.isOverlapping = false
        })
        this.resetSImagePosition()
      }
    },
    setSImagePosition(x: number, y: number, svgRect: DOMRect) {
      const sImageRect = (this.$refs.sImage as SVGImageElement).getBoundingClientRect()
      this.sImagePosition.x = x - (sImageRect.width / 2) - svgRect.x
      this.sImagePosition.y = y - (sImageRect.height / 2) - svgRect.y
      this.sImagePosition.width = sImageRect.width
      this.sImagePosition.height = sImageRect.height
    },
    resetSImagePosition() {
      this.sImagePosition = { x: 0, y: 0, width: 0, height: 0 }
    },
    handleElementClick(element: any) {
      // 处理元素点击事件
    }
  },
  mounted() {
    // 初始化S图片的位置
    const sImageRect = (this.$refs.sImage as SVGImageElement).getBoundingClientRect()
    this.sImagePosition.width = sImageRect.width
    this.sImagePosition.height = sImageRect.height
  }
})
</script>

<style scoped>
.svg-container {
  position: relative;
  width: 100%;
  height: 100%;
}
</style>