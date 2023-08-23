<template>
  <div>{{ Math.round(rotateAngle) }},  {{ tempAngle }}</div>
  <div class="draw-board">
    <svg style="background-color: white;" ref="svgRef" @mousedown="handleMouseDown(null, $event)"
      @mousemove="handleMouseMove" @mouseup="handleMouseUp" :width="svgWidth" :height="svgHeight">
      <g :transform="getRotateValue">
        <!-- 添加选框边框 -->
        <rect v-if="selectedRectIndex == -1" :x="selectedRectX" :y="selectedRectY" :width="selectedRectWidth"
          :height="selectedRectHeight" fill="none" stroke="black" />

        <!-- 添加缩放手柄 -->
        <rect v-if="selectedRectIndex !== -1" class="handle-top-left" :x="selectedRectX - handleSize / 2"
          :y="selectedRectY - handleSize / 2" :width="handleSize" :height="handleSize" fill="white" stroke="black"
          @mousedown="handleMouseDown('tl', $event)" @mouseup="handleMouseUp" />
        <rect v-if="selectedRectIndex !== -1" class="handle-top-right"
          :x="selectedRectX + selectedRectWidth - handleSize / 2" :y="selectedRectY - handleSize / 2" :width="handleSize"
          :height="handleSize" fill="white" stroke="black" @mousedown="handleMouseDown('tr', $event)"
          @mouseup="handleMouseUp" />
        <rect v-if="selectedRectIndex !== -1" class="handle-bottom-left" :x="selectedRectX - handleSize / 2"
          :y="selectedRectY + selectedRectHeight - handleSize / 2" :width="handleSize" :height="handleSize" fill="white"
          stroke="black" @mousedown="handleMouseDown('bl', $event)" @mouseup="handleMouseUp" />
        <rect v-if="selectedRectIndex !== -1" class="handle-bottom-right"
          :x="selectedRectX + selectedRectWidth - handleSize / 2" :y="selectedRectY + selectedRectHeight - handleSize / 2"
          :width="handleSize" :height="handleSize" fill="white" stroke="black" @mousedown="handleMouseDown('br', $event)"
          @mouseup="handleMouseUp" />

        <!-- 旋转手柄 -->
        <line v-if="selectedRectIndex !== -1" class="handle-rotate-line" :x1="selectedRectX + selectedRectWidth / 2"
          :y1="selectedRectY - rotateHandleLength" :x2="selectedRectX + selectedRectWidth / 2" :y2="selectedRectY"
          stroke="black" />

        <circle v-if="selectedRectIndex !== -1" class="handle-rotate-circle" :cx="selectedRectX + selectedRectWidth / 2"
          :cy="selectedRectY - rotateHandleLength" :r="rotateHandleRadius" fill="white" stroke="black"
          @mousedown="handleMouseDown('rotate', $event)" @mouseup="handleMouseUp" />
      </g>

      <!-- 添加元素 -->
      <rect :transform="getRotateValue" v-for="(rect, index) in rects" :key="index" :x="rect.x" :y="rect.y"
        :width="rect.width" :height="rect.height" :fill="rect.color" :stroke-width="isSelected(index) ? 2 : 0"
        stroke="blue" @mousemove="handleMouseMove" />

      <!-- 添加预测缩放和旋转的箭头 -->
      <line v-if="selectedRectIndex !== -1 && showPrediction" class="prediction-line" :x1="predictionX" :y1="predictionY"
        :x2="mouseX" :y2="mouseY" stroke="black" />
      <path v-if="selectedRectIndex !== -1 && showPrediction" class="prediction-arrow" :d="predictionArrowPath"
        fill="black" />
    </svg>
  </div>
</template>

<script lang="ts">
import { defineComponent, reactive } from 'vue';

interface Rect {
  x: number;
  y: number;
  width: number;
  height: number;
  color: string;
}

type SelectedRectFunction = 'tl' | 'tr' | 'bl' | 'br' | 'rotate' | null

export default defineComponent({
  data() {
    return {
      svgWidth: 400,
      svgHeight: 400,
      mouseX: 0,
      mouseY: 0,
      rects: reactive<Rect[]>([]),
      handleSize: 12, // 手柄大小
      rotateAngle: 0, // 旋转角度
      tempAngle: 0,
      rotateHandleLength: 20, // 旋转线的长度
      rotateHandleRadius: 8, // 旋转圆的半径
      selectedRectIndex: -1,
      selectedRectStart: { x: 0, y: 0, width: 0, height: 0 },
      selectedRectCenter: { x_old: 0, y_old: 0 },
      isDragging: false,
      isSelectedRect: false,
      SelectedRectPostion: null as SelectedRectFunction,
    }
  },
  mounted() {
    this.createRect(100, 100);
  },
  methods: {
    handleMouseDown(position: SelectedRectFunction, event: MouseEvent) {
      const { clientX, clientY } = event;
      const svg = this.$refs.svgRef as SVGSVGElement;
      if (svg && position == null) {
        const svgRect = svg.getBoundingClientRect();
        this.mouseX = clientX - svgRect.left;
        this.mouseY = clientY - svgRect.top;
        // const clickedIndex = this.rects.reduceRight((acc, rect, index) => {
        //   if (
        //     this.mouseX >= rect.x &&
        //     this.mouseX <= rect.x + rect.width &&
        //     this.mouseY >= rect.y &&
        //     this.mouseY <= rect.y + rect.height &&
        //     acc === -1
        //   ) {
        //     return index;
        //   }
        //   return acc;
        // }, -1);
        const clickedIndex = this.rects.reduceRight((acc, rect, index) => {
          // 获取<rect>元素的旋转角度
          const rotateAngle = rect.rotateAngle;

          // 校正鼠标位置为相对于旋转前的<rect>元素的坐标
          const centerX = rect.x + rect.width / 2;
          const centerY = rect.y + rect.height / 2;
          const deltaX = this.mouseX - centerX;
          const deltaY = this.mouseY - centerY;
          const rotatedX = centerX + deltaX * Math.cos(-rotateAngle) - deltaY * Math.sin(-rotateAngle);
          const rotatedY = centerY + deltaX * Math.sin(-rotateAngle) + deltaY * Math.cos(-rotateAngle);
          // console.log("cx: " + centerX.toFixed(0), "cy: " + centerY.toFixed(0), 
          // "mx: "+this.mouseX.toFixed(0), "my: " + this.mouseY.toFixed(0),
          // "rx: "+rotatedX.toFixed(0), "ro: " + rotatedY.toFixed(0))
          if (
            rotatedX >= rect.x &&
            rotatedX <= rect.x + rect.width &&
            rotatedY >= rect.y &&
            rotatedY <= rect.y + rect.height &&
            acc === -1
          ) {
            return index;
          }
          return acc;
        }, -1);
        if (clickedIndex !== -1 && position == null && this.SelectedRectPostion == null) {
          this.$nextTick(() => {
            this.selectedRectIndex = clickedIndex;
            this.selectedRectStart.y = this.selectedRectY;
            this.selectedRectStart.x = this.selectedRectX;
            this.selectedRectStart.width = this.selectedRectWidth;
            this.selectedRectStart.height = this.selectedRectHeight;
            this.isDragging = true;
          });
        } else if (this.SelectedRectPostion == null) {  //选中背景，取消选框
          this.selectedRectIndex = -1;
        }

      } else if (position != null) {
        this.selectedRectStart.y = this.selectedRectY;
        this.selectedRectStart.x = this.selectedRectX;
        this.selectedRectStart.width = this.selectedRectWidth;
        this.selectedRectStart.height = this.selectedRectHeight;
        this.selectedRectCenter = { x_old: this.getNowCentor.x, y_old: this.getNowCentor.y };
        this.isSelectedRect = true;
        this.isDragging = false;
        this.SelectedRectPostion = position;
      }
    },
    handleMouseMove(event: MouseEvent) {
      const svg = this.$refs.svgRef as SVGSVGElement;

      //处理拖拽移动
      if (this.isDragging && svg) {
        const { clientX, clientY } = event;
        const svgRect = svg.getBoundingClientRect();
        const dragDeltaX = clientX - svgRect.left - this.mouseX;
        const dragDeltaY = clientY - svgRect.top - this.mouseY;

        if (this.selectedRectIndex !== -1) {
          this.rects[this.selectedRectIndex].x = this.selectedRectStart.x + dragDeltaX;
          this.rects[this.selectedRectIndex].y = this.selectedRectStart.y + dragDeltaY;
        }
      } else if (this.isSelectedRect && svg) {
        //处理旋转
        if (this.SelectedRectPostion == 'rotate') {
          const rectCenterX = this.getNowCentor.x;
          const rectCenterY = this.getNowCentor.y;

          const deltaX = event.clientX - rectCenterX;
          const deltaY = event.clientY - rectCenterY;
          const angle = Math.atan2(deltaY, deltaX) * (180 / Math.PI);

          this.rotateAngle = angle - 90;
          return;
        }
        //处理缩放
        const { clientX, clientY } = event;
        const svgRect = svg.getBoundingClientRect();
        const dragDeltaX = clientX - svgRect.left - this.mouseX;
        const dragDeltaY = clientY - svgRect.top - this.mouseY;

        let tempSelectedRectStart = { x: this.selectedRectStart.x, y: this.selectedRectStart.y };
        let newWidth = 0;
        let newHeight = 0;
        switch (this.SelectedRectPostion) {
          case 'tl':
            newWidth = this.selectedRectStart.width - dragDeltaX;
            newHeight = this.selectedRectStart.height - dragDeltaY;
            // 根据左上角位置调整左上角坐标
            tempSelectedRectStart.x += dragDeltaX;
            tempSelectedRectStart.y += dragDeltaY;
            break;
          case 'tr':
            newWidth = this.selectedRectStart.width + dragDeltaX;
            newHeight = this.selectedRectStart.height - dragDeltaY;
            // 根据右上角位置调整左上角坐标
            tempSelectedRectStart.y += dragDeltaY;
            break;
          case 'bl':
            newWidth = this.selectedRectStart.width - dragDeltaX;
            newHeight = this.selectedRectStart.height + dragDeltaY;
            // 根据左下角位置调整左上角坐标
            tempSelectedRectStart.x += dragDeltaX;
            break;
          case 'br':
            newWidth = this.selectedRectStart.width + dragDeltaX;
            newHeight = this.selectedRectStart.height + dragDeltaY;
            break;
          default:
            break;
        }

        if (newWidth > 0 && newHeight > 0) {
          const centerX = tempSelectedRectStart.x + newWidth / 2;
          const centerY = tempSelectedRectStart.y + newHeight / 2;

          const angle = this.rotateAngle * (Math.PI / 180);
          const cosAngle = Math.cos(angle);
          const sinAngle = Math.sin(angle);

          const dx = centerX - this.selectedRectCenter.x_old;
          const dy = centerY - this.selectedRectCenter.y_old;

          const rotatedDX = dx * cosAngle - dy * sinAngle;
          const rotatedDY = dx * sinAngle + dy * cosAngle;

          const newX = this.selectedRectCenter.x_old + rotatedDX;
          const newY = this.selectedRectCenter.y_old + rotatedDY;

          this.rects[this.selectedRectIndex].x = newX - newWidth / 2;
          this.rects[this.selectedRectIndex].y = newY - newHeight / 2;
          this.rects[this.selectedRectIndex].width = newWidth;
          this.rects[this.selectedRectIndex].height = newHeight;
        }
      }
    },
    handleMouseUp() {
      this.isDragging = false;
      this.isSelectedRect = false;
      this.SelectedRectPostion = null

      this.selectedRectStart.x = this.rects[this.selectedRectIndex].x;
      this.selectedRectStart.y = this.rects[this.selectedRectIndex].y;
    },
    isSelected(index: number) {
      return index === this.selectedRectIndex;
    },
    createRect(x: number, y: number) {
      const width = Math.random() * 50 + 50;
      const height = Math.random() * 50 + 50;
      const color = '#' + Math.random().toString(16).substr(-6);
      this.selectedRectIndex = 0
      // this.rotateAngle = 90
      this.rects.push({ x, y, width, height, color });
    },
  },
  computed: {
    getRotateValue(): string {
      return 'rotate(' + this.rotateAngle + ' ' + this.getNowCentor.x + ' ' + this.getNowCentor.y + ')';
    },
    getNowCentor(): { x: number, y: number } {
      return {
        x: this.selectedRectX + this.selectedRectWidth / 2,
        y: this.selectedRectY + this.selectedRectHeight / 2
      }
    },
    selectedRectX(): number {
      if (this.selectedRectIndex !== -1) {
        return this.rects[this.selectedRectIndex].x;
      }
      return 0;
    },
    selectedRectY(): number {
      if (this.selectedRectIndex !== -1) {
        return this.rects[this.selectedRectIndex].y;
      }
      return 0;
    },
    selectedRectWidth(): number {
      if (this.selectedRectIndex !== -1) {
        return this.rects[this.selectedRectIndex].width;
      }
      return 0;
    },
    selectedRectHeight(): number {
      if (this.selectedRectIndex !== -1) {
        return this.rects[this.selectedRectIndex].height;
      }
      return 0;
    },
    showPrediction(): boolean {
      return this.isDragging && this.selectedRectIndex !== -1;
    },
    predictionX(): number {
      if (this.showPrediction) {
        return this.selectedRectX + this.selectedRectWidth / 2;
      }
      return 0;
    },
    predictionY(): number {
      if (this.showPrediction) {
        return this.selectedRectY + this.selectedRectHeight / 2;
      }
      return 0;
    },
    predictionArrowPath(): string {
      if (this.showPrediction) {
        const dx = this.mouseX - this.predictionX;
        const dy = this.mouseY - this.predictionY;
        const angle = Math.atan2(dy, dx);
        const arrowSize = this.handleSize * 2;
        const arrowHalfSize = arrowSize / 2;
        const headAngle = Math.PI / 6;

        const arrowX = this.predictionX + Math.cos(angle) * arrowHalfSize;
        const arrowY = this.predictionY + Math.sin(angle) * arrowHalfSize;

        const arrowLeftX = arrowX + Math.cos(angle - headAngle) * arrowHalfSize;
        const arrowLeftY = arrowY + Math.sin(angle - headAngle) * arrowHalfSize;

        const arrowRightX = arrowX + Math.cos(angle + headAngle) * arrowHalfSize;
        const arrowRightY = arrowY + Math.sin(angle + headAngle) * arrowHalfSize;

        return `M${arrowX},${arrowY}L${arrowLeftX},${arrowLeftY}L${arrowRightX},${arrowRightY}Z`;
      }
      return '';
    },
  },
});
</script>

<style scoped>
.draw-board {
  position: relative;
}

.handle-top-left,
.handle-top-right,
.handle-bottom-left,
.handle-bottom-right {
  cursor: pointer;
}

.prediction-line {
  stroke-dasharray: 5;
  pointer-events: none;
}

.prediction-arrow {
  transform-origin: center;
}
</style>
