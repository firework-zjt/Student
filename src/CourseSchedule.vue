<template>
  <!-- 原模板代码保持不变 -->
  <div class="course-schedule-container">
    <!-- Left side: Subject list -->
    <div class="subject-list" :style="{ width: subjectListWidth }">
      <h3>{{ SubjectTitle }}</h3>
      <ul class="datalist">
        <li 
          v-for="(subject, index) in subjects" 
          :key="index"
          :class="{ active: selectedSubject === subject }"
          @click="selectedSubject = subject"
          draggable="true"
          @dragstart="onDragStart($event, subject)"
        >
          {{ subject }}
        </li>
      </ul>
    </div>
    
    <!-- Right side: Course schedule grid -->
    <div class="schedule-container" :style="{ width: scheduleContainerWidth }">
      <div class="schedule-header">
        <h3>{{ mainTitle }}</h3>
        <div class="action-buttons">
          <button @click="exportAsImage" class="export-btn">导出图片</button>
          <button @click="clearAll" class="clear-btn">清空</button>
        </div>
      </div>
      
      <div ref="scheduleGrid" class="schedule-grid">
        <!-- Header row -->
        <div class="grid-row header-row">
          <div class="grid-cell header-cell"></div>
          <div 
            v-for="(day, index) in dayslist" 
            :key="'day-' + index" 
            class="grid-cell header-cell"
            :style="{ width: cellWidth, height: cellHeight }"
          >
            {{ day }}
          </div>
        </div>
        
        <!-- Time slots rows -->
        <div 
          v-for="(time, rowIndex) in times" 
          :key="'time-' + rowIndex" 
          class="grid-row"
        >
          <div class="grid-cell time-cell" :style="{ width: cellWidth, height: cellHeight }">{{ time }}</div>
          <div 
            v-for="(day, colIndex) in dayslist" 
            :key="'cell-' + rowIndex + '-' + colIndex" 
            class="grid-cell"
            :style="{ width: cellWidth, height: cellHeight }"
            :class="{ 'drag-over': isDragOver(rowIndex, colIndex) }"
            @dragenter.prevent="onDragEnter(rowIndex, colIndex)"
            @dragleave.prevent="onDragLeave(rowIndex, colIndex)"
            @dragover.prevent="onDragOver($event)"
            @drop="onDrop($event, rowIndex, colIndex)"
            draggable="true"
            @dragstart="onCellDragStart($event, rowIndex, colIndex)"
            @dragend="onDragEnd($event, rowIndex, colIndex)"
            @click="addCourseOnClick(rowIndex, colIndex)"
          >
            <div v-if="schedule[rowIndex] && schedule[rowIndex][colIndex]" class="course-item">
              {{ schedule[rowIndex][colIndex] }}
            </div>
          </div>
        </div>
        
        <!-- Watermark (hidden until export) -->
        <div class="watermark" ref="watermark" v-show="false">
          {{ watermarkText }}
        </div>
      </div>
    </div>
    <!-- 移动成功提示 -->
    <div v-if="showMoveSuccess" class="move-success-tip">
      <p>移动成功</p>
    </div>
    <!-- 确认对话框 -->
    <div v-if="showConfirmDialog" class="confirm-dialog">
      <div class="dialog-content">
        <p>确认移除该课程吗？</p>
        <div class="dialog-buttons">
          <button @click="confirmRemove" class="confirm-btn">确认</button>
          <button @click="cancelRemove" class="cancel-btn">取消</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import html2canvas from 'html2canvas';

/**
 * 课程拖拽时间表组件，支持拖拽课程到指定单元格，导出课程表为图片等功能。
 * 
 * @component
 * @name CourseSchedule
 * @author Fireworks
 * @prop {Array} subjects - 课程列表
 * @prop {Array} times - 时间表
 * @prop {Array} dayslist - 星期列表
 * @prop {string} watermarkText - 水印文本
 * @prop {string} mainTitle - 主标题
 * @prop {string} SubjectTitle - 课程列表标题
 * @prop {string} primaryColor - 主体颜色
 * @prop {string} subjectListWidth - 左侧课程列表宽度
 * @prop {string} scheduleContainerWidth - 右侧课程表容器宽度
 * @prop {string} cellWidth - 右侧课程表内单元格宽度
 * @prop {string} cellHeight - 右侧课程表内单元格高度
 * @emits course-added - 课程添加事件
 * @emits course-removed - 课程移除事件
 * @emits export-complete - 导出完成事件
 * @emits clear-complete - 清空完成事件
 * @emits move-success - 移动成功事件
 * @emits confirm-remove - 确认移除事件
 * @emits cancel-remove - 取消移除事件
 * @emits onDragStart - 开始拖拽事件
 * @emits onCellDragStart - 开始拖拽单元格事件
 * @example
 */
export default {
  name: 'CourseSchedule',
  props: {
    /**
     * 课程列表
     * @type {Array}
     * @default ['数学', '英语', '语文']
     */
    subjects: {
      type: Array,
      default: () => []
    },
    /**
     * 时间表
     * @type {Array}
     * @default ['第一节', '第二节', '第三节', '第四节', '第五节', '第六节', '第七节', '第八节', '第九节']
     */
    times: {
      type: Array,
      default: () => ['第一节', '第二节', '第三节', '第四节', '第五节', '第六节']
    },
    /**
     * 星期列表
     * @type {Array}
     * @default ['周一', '周二', '周三', '周四', '周五']
     */
    dayslist: {
      type: Array,
      default: () => ['周一', '周二', '周三', '周四', '周五'] 
    },
    /**
     * 水印文本
     * @type {string}
     * @default 'Fireworks的课程表组件'
     */
    watermarkText: {
      type: String,
      default: 'Fireworks的课程表组件'
    },
    /**
     * 主标题
     * @type {string}
     * @default '课程表'
     */
    mainTitle: {
      type: String,
      default: '课程表'
    },
    /**
     * 课程列表标题
     * @type {string}
     * @default '课程列表'
     */
    SubjectTitle: {
      type: String,
      default: '课程列表'
    },
    /**
     * 主体颜色
     * @type {string}
     * @default '#4299e1'
     */
    primaryColor: {
      type: String,
      default: '#ff4100'
    },
    /**
     * 左侧课程列表宽度
     * @type {string}
     * @default '220px'
     */
    subjectListWidth: {
      type: String,
      default: '220px'
    },
    /**
     * 右侧课程表容器宽度
     * @type {string}
     * @default 'auto'
     */
    scheduleContainerWidth: {
      type: String,
      default: 'auto'
    },
    /**
     * 右侧课程表内单元格宽度
     * @type {string}
     * @default 'auto'
     */
    cellWidth: {
      type: String,
      default: 'auto'
    },
    /**
     * 右侧课程表内单元格高度
     * @type {string}
     * @default '90px'
     */
    cellHeight: {
      type: String,
      default: '90px'
    }
  },
  data() {
    return {
      selectedSubject: '',
      schedule: [],
      dragTarget: null,
      dragSource: null, // 用于记录拖拽的源单元格
      showMoveSuccess: false, // 控制移动成功提示的显示
      showConfirmDialog: false, // 控制确认对话框的显示
      confirmRow: null,
      confirmCol: null
    };
  },
  created() {
    if (this.subjects.length > 0) {
      this.selectedSubject = this.subjects[0];
    }
    this.initializeSchedule();
  },
  methods: {
    /**
     * 初始化课程表
     */
    initializeSchedule() {
      this.schedule = [];
      for (let i = 0; i < this.times.length; i++) {
        this.schedule[i] = new Array(this.dayslist.length).fill('');
      }
    },
    /**
     * 添加课程到指定单元格
     * @param {number} row - 行索引
     * @param {number} col - 列索引
     */
    addCourse(row, col) {
      if (this.selectedSubject) {
        this.$set(this.schedule[row], col, this.selectedSubject);
        this.$emit('course-added', { row, col, subject: this.selectedSubject });
      }
    },
    /**
     * 点击单元格添加课程
     * @param {number} row - 行索引
     * @param {number} col - 列索引
     */
    addCourseOnClick(row, col) {
      if (this.selectedSubject && !this.schedule[row][col]) {
        this.addCourse(row, col);
        this.showMoveSuccess = true;
        setTimeout(() => {
          this.showMoveSuccess = false;
        }, 1000);
      }
    },
    /**
     * 开始拖拽课程
     * @param {Event} event - 拖拽事件
     * @param {string} subject - 课程名称
     */
    onDragStart(event, subject) {
      event.dataTransfer.setData('text/plain', subject);
      event.dataTransfer.effectAllowed = 'move';
      const dragImage = document.createElement('div');
      dragImage.textContent = subject;
      // 使用 CSS 变量设置背景颜色和字体颜色
      dragImage.style.backgroundColor = 'var(--drag-bg-color)';
      dragImage.style.padding = '10px';
      dragImage.style.borderRadius = '6px';
      dragImage.style.color = 'var(--primary-color)';
      dragImage.style.fontWeight = '500';
      dragImage.style.position = 'absolute';
      dragImage.style.top = '-1000px';
      document.body.appendChild(dragImage);
      event.dataTransfer.setDragImage(dragImage, 35, 35);
      this.selectedSubject = subject;
      setTimeout(() => {
        document.body.removeChild(dragImage);
      }, 0);
    },
    /**
     * 开始拖拽单元格
     * @param {Event} event - 拖拽事件
     * @param {number} row - 行索引
     * @param {number} col - 列索引
     */
    onCellDragStart(event, row, col) {
      const subject = this.schedule[row][col];
      if (subject) {
        event.dataTransfer.setData('text/plain', subject);
        event.dataTransfer.effectAllowed = 'move';
        this.dragSource = { row, col }; // 记录拖拽的源单元格
      } else {
        event.preventDefault(); // 如果单元格为空，阻止拖拽
      }
    },
    /**
     * 拖拽进入单元格
     * @param {number} row - 行索引
     * @param {number} col - 列索引
     */
    onDragEnter(row, col) {
      this.dragTarget = { row, col };
    },
    /**
     * 拖拽离开单元格
     * @param {number} row - 行索引
     * @param {number} col - 列索引
     */
    onDragLeave(row, col) {
      if (this.dragTarget && this.dragTarget.row === row && this.dragTarget.col === col) {
        this.dragTarget = null;
      }
    },
    /**
     * 拖拽经过单元格
     * @param {Event} event - 拖拽事件
     */
    onDragOver(event) {
      event.preventDefault();
      event.dataTransfer.dropEffect = 'move';
    },
    /**
     * 判断是否正在拖拽到指定单元格
     * @param {number} row - 行索引
     * @param {number} col - 列索引
     * @returns {boolean} - 是否正在拖拽到指定单元格
     */
    isDragOver(row, col) {
      return this.dragTarget && this.dragTarget.row === row && this.dragTarget.col === col;
    },
    /**
     * 放下拖拽的课程
     * @param {Event} event - 拖拽事件
     * @param {number} row - 行索引
     * @param {number} col - 列索引
     */
    onDrop(event, row, col) {
      event.preventDefault();
      const subject = event.dataTransfer.getData('text/plain');
      if (subject) {
        const tempElement = document.createElement('div');
        tempElement.className = 'course-item';
        tempElement.textContent = subject;
        tempElement.style.position = 'absolute';
        tempElement.style.zIndex = '100';
        tempElement.style.opacity = '0.8';
        // 使用 CSS 变量设置背景颜色
        tempElement.style.backgroundColor = 'var(--drag-bg-color)';
        const cell = event.currentTarget;
        const cellRect = cell.getBoundingClientRect();
        tempElement.style.left = `${event.clientX - 50}px`;
        tempElement.style.top = `${event.clientY - 25}px`;
        tempElement.style.width = '100px';
        tempElement.style.height = '50px';
        document.body.appendChild(tempElement);
        tempElement.style.transition = 'all 0.3s ease-out';
        setTimeout(() => {
          tempElement.style.left = `${cellRect.left}px`;
          tempElement.style.top = `${cellRect.top}px`;
          tempElement.style.width = `${cellRect.width}px`;
          tempElement.style.height = `${cellRect.height}px`;
          tempElement.style.opacity = '0';
          setTimeout(() => {
            document.body.removeChild(tempElement);
            // 直接覆盖目标单元格内容
            this.$set(this.schedule[row], col, subject);
            if (this.dragSource) {
              if (this.dragSource.row === row && this.dragSource.col === col) {
                // 拖拽到同一个单元格，不做处理
              } else {
                this.$set(this.schedule[this.dragSource.row], this.dragSource.col, ''); // 清除源单元格内容
                this.showMoveSuccess = true;
                setTimeout(() => {
                  this.showMoveSuccess = false;
                }, 1000);
              }
              this.dragSource = null; // 清除拖拽源
            }
            this.$emit('course-added', { row, col, subject });
            this.dragTarget = null;
          }, 300);
        }, 10);
      }
    },
    /**
     * 拖拽结束
     * @param {Event} event - 拖拽事件
     * @param {number} row - 行索引
     * @param {number} col - 列索引
     */
    onDragEnd(event, row, col) {
      const gridRect = this.$refs.scheduleGrid.getBoundingClientRect();
      const x = event.clientX;
      const y = event.clientY;
      if (
        x < gridRect.left ||
        x > gridRect.right ||
        y < gridRect.top ||
        y > gridRect.bottom
      ) {
        this.showConfirmDialog = true;
        this.confirmRow = row;
        this.confirmCol = col;
      }
    },
    /**
     * 确认移除课程
     */
    confirmRemove() {
      this.$set(this.schedule[this.confirmRow], this.confirmCol, '');
      this.showConfirmDialog = false;
      this.$emit('course-removed', { row: this.confirmRow, col: this.confirmCol });
    },
    /**
     * 取消移除课程
     */
    cancelRemove() {
      this.showConfirmDialog = false;
    },
    /**
     * 导出课程表为图片
     */
    async exportAsImage() {
      try {
        const element = this.$refs.scheduleGrid;
        const gridRect = element.getBoundingClientRect();
        if (this.$refs.watermark) {
          this.$refs.watermark.style.display = 'none';
        }
        const watermarkContainer = document.createElement('div');
        watermarkContainer.style.position = 'absolute';
        watermarkContainer.style.top = '0';
        watermarkContainer.style.left = '0';
        watermarkContainer.style.width = '100%';
        watermarkContainer.style.height = '100%';
        watermarkContainer.style.overflow = 'hidden';
        watermarkContainer.style.pointerEvents = 'none';
        watermarkContainer.style.zIndex = '10';
        const spacingX = 300;
        const spacingY = 240;
        const cols = Math.ceil(gridRect.width / spacingX) + 1;
        const rows = Math.ceil(gridRect.height / spacingY) + 1;
        for (let row = -1; row < rows; row++) {
          for (let col = -1; col < cols; col++) {
            const watermark = document.createElement('div');
            watermark.textContent = this.watermarkText;
            watermark.style.position = 'absolute';
            watermark.style.left = `${col * spacingX}px`;
            watermark.style.top = `${row * spacingY}px`;
            watermark.style.fontSize = '24px';
            // 使用 CSS 变量设置水印颜色
            watermark.style.color = 'var(--watermark-color)';
            watermark.style.whiteSpace = 'nowrap';
            watermark.style.transform = 'rotate(30deg)';
            watermark.style.transformOrigin = 'center center';
            watermark.style.userSelect = 'none';
            watermarkContainer.appendChild(watermark);
          }
        }
        element.appendChild(watermarkContainer);
        const canvas = await html2canvas(element, {
          backgroundColor: '#ffffff',
          scale: 2,
          logging: false
        });
        element.removeChild(watermarkContainer);
        const image = canvas.toDataURL('image/png');
        const link = document.createElement('a');
        link.download = '课程表.png';
        link.href = image;
        link.click();
        this.$emit('export-complete', image);
      } catch (error) {
        console.error('导出图片失败:', error);
      }
    },
    /**
     * 清空所有单元格内容
     */
    clearAll() {
      this.initializeSchedule();
    },
  },
  mounted() {
    this.$el.style.setProperty('--primary-color', this.primaryColor);
    // 计算拖拽背景颜色和水印颜色
    const primaryColor = this.primaryColor;
    const lightenedColor = lightenColor(primaryColor, 0.8);
    const watermarkColor = `rgba(${hexToRgb(primaryColor).join(',')}, 0.3)`;
    this.$el.style.setProperty('--drag-bg-color', lightenedColor);
    this.$el.style.setProperty('--watermark-color', watermarkColor);

    // 计算课程列表背景颜色和课程行列背景颜色
    const subjectListBgColor = lightenColor(primaryColor, 0.9);
    const courseRowColBgColor = lightenColor(primaryColor, 0.95);

    // 计算鼠标悬停加深颜色
    const dragBgHoverColor = lightenColor(primaryColor, 0.7);
    const subjectListBgHoverColor = lightenColor(primaryColor, 0.8);
    const courseRowColBgHoverColor = lightenColor(primaryColor, 0.9);

    // 设置新的 CSS 变量
    this.$el.style.setProperty('--subject-list-bg-color', subjectListBgColor);
    this.$el.style.setProperty('--course-row-col-bg-color', courseRowColBgColor);
    this.$el.style.setProperty('--drag-bg-hover-color', dragBgHoverColor);
    this.$el.style.setProperty('--subject-list-bg-hover-color', subjectListBgHoverColor);
    this.$el.style.setProperty('--course-row-col-bg-hover-color', courseRowColBgHoverColor);
  },
  beforeUnmount() {
    // 移除 CSS 变量
    this.$el.style.removeProperty('--primary-color');
    this.$el.style.removeProperty('--drag-bg-color');
    this.$el.style.removeProperty('--watermark-color');
    this.$el.style.removeProperty('--subject-list-bg-color');
    this.$el.style.removeProperty('--course-row-col-bg-color');
    this.$el.style.removeProperty('--drag-bg-hover-color');
    this.$el.style.removeProperty('--subject-list-bg-hover-color');
    this.$el.style.removeProperty('--course-row-col-bg-hover-color');
  }
}

// 辅助函数：将十六进制颜色转换为 RGB 数组
function hexToRgb(hex) {
  const r = parseInt(hex.slice(1, 3), 16);
  const g = parseInt(hex.slice(3, 5), 16);
  const b = parseInt(hex.slice(5, 7), 16);
  return [r, g, b];
}

// 辅助函数：提亮颜色
function lightenColor(hex, factor) {
  const [r, g, b] = hexToRgb(hex);
  const newR = Math.min(255, Math.round(r + (255 - r) * factor));
  const newG = Math.min(255, Math.round(g + (255 - g) * factor));
  const newB = Math.min(255, Math.round(b + (255 - b) * factor));
  return `#${newR.toString(16).padStart(2, '0')}${newG.toString(16).padStart(2, '0')}${newB.toString(16).padStart(2, '0')}`;
}
</script>

<style scoped>
/* 使用 CSS 变量设置主体颜色 */
:root {
  --primary-color: #4299e1;
  --drag-bg-color: #ebf8ff;
  --watermark-color: rgba(66, 153, 225, 0.3);
  --subject-list-bg-color: #f7faff; /* 新增 */
  --course-row-col-bg-color: #f0f7ff; /* 新增 */
  --drag-bg-hover-color: #d6eaf8; /* 新增 */
  --subject-list-bg-hover-color: #e0f0ff; /* 新增 */
  --course-row-col-bg-hover-color: #d0e5ff; /* 新增 */
}

/* 样式保持不变，部分颜色使用 CSS 变量 */
.course-schedule-container {
  display: flex;
  font-family: 'Segoe UI', 'Microsoft YaHei', sans-serif;
  border: none;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
  background-color: #ffffff;
}

.subject-list {
  padding: 20px;
  border-right: 1px solid #e6f0ff;
}
.subject-list h3 {
  margin-top: 0;
  margin-bottom: 20px;
  color: #2c5282;
  font-weight: 600;
  font-size: 18px;
  letter-spacing: 0.5px;
}

.datalist {
  list-style: none;
  padding: 0;
  margin: 0;
}

.datalist li {
  padding: 12px 16px;
  margin-bottom: 8px;
  background-color: #fff;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s ease;
  user-select: none;
  color: #4a5568;
  border: 1px solid #edf2f7;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.02);
}

.datalist li:hover {
  background-color: var(--drag-bg-hover-color);
  transform: translateY(-1px);
}

.datalist li.active {
  background-color: var(--drag-bg-color);
  border-left: 3px solid var(--primary-color);
  color: var(--primary-color);
  font-weight: 500;
}

.datalist li[draggable=true] {
  cursor: grab;
}

.datalist li[draggable=true]:active {
  cursor: grabbing;
}

.schedule-container {
  flex: 1;
  padding: 20px;
  background-color: #fff;
}

.schedule-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.schedule-header h3 {
  margin: 0;
  color: #2c5282;
  font-weight: 600;
  font-size: 18px;
  letter-spacing: 0.5px;
}

.action-buttons {
  display: flex;
  gap: 10px;
}

.export-btn, .clear-btn {
  background-color: var(--primary-color);
  color: white;
  border: none;
  padding: 10px 18px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 500;
  transition: all 0.2s ease;
  box-shadow: 0 2px 4px rgba(66, 153, 225, 0.2);
}

.export-btn:hover, .clear-btn:hover {
  background-color: #3182ce;
  transform: translateY(-1px);
  box-shadow: 0 4px 6px rgba(66, 153, 225, 0.25);
}

.export-btn:active, .clear-btn:active {
  transform: translateY(0);
}

.schedule-grid {
  position: relative;
  border: 1px solid #e6f0ff;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.03);
}

.grid-row {
  display: flex;
}

.grid-cell {
  flex: 1;
  border: 1px solid #e6f0ff;
  padding: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  text-align: center;
  transition: background-color 0.2s ease;
}

.grid-cell:hover {
  background-color: var(--course-row-col-bg-hover-color);
}

.header-row {
  background-color: var(--course-row-col-bg-color); /* 修改 */
}

.header-row .grid-cell:hover {
  background-color: var(--course-row-col-bg-hover-color);
}

.header-cell {
  min-height: 50px;
  font-weight: 600;
  color: #2c5282;
  letter-spacing: 0.5px;
}

.time-cell {
  background-color: var(--course-row-col-bg-color); /* 修改 */
}

.time-cell:hover {
  background-color: var(--course-row-col-bg-hover-color);
}

.course-item {
  width: 100%;
  height: 100%;
  background-color: var(--drag-bg-color);
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 8px;
  color: var(--primary-color);
  font-weight: 500;
  box-shadow: 0 2px 5px rgba(66, 153, 225, 0.1);
  transition: all 0.2s ease;
}

.course-item:hover {
  background-color: var(--drag-bg-hover-color);
  transform: scale(1.02);
}

.watermark {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 36px;
  color: var(--watermark-color);
  pointer-events: none;
  white-space: nowrap;
  user-select: none;
  transform: rotate(-30deg);
  z-index: 10;
}

.grid-cell.drag-over {
  background-color: var(--drag-bg-color);
  border: 2px dashed var(--primary-color);
}

/* 移动成功提示样式 */
.move-success-tip {
  position: fixed;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  background-color: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 10px 20px;
  border-radius: 6px;
  z-index: 100;
}

/* 确认对话框样式 */
.confirm-dialog {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 200;
}

.dialog-content {
  background-color: white;
  padding: 20px;
  border-radius: 6px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
}

.dialog-buttons {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-top: 20px;
}

.confirm-btn, .cancel-btn {
  background-color: var(--primary-color);
  color: white;
  border: none;
  padding: 10px 18px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 500;
  transition: all 0.2s ease;
  box-shadow: 0 2px 4px rgba(66, 153, 225, 0.2);
}

.confirm-btn:hover, .cancel-btn:hover {
  background-color: #3182ce;
  transform: translateY(-1px);
  box-shadow: 0 4px 6px rgba(66, 153, 225, 0.25);
}

.confirm-btn:active, .cancel-btn:active {
  transform: translateY(0);
}

@media (max-width: 768px) {
  .course-schedule-container {
    flex-direction: column;
  }
  
  .subject-list {
    width: 100%;
    border-right: none;
    border-bottom: 1px solid #e6f0ff;
  }
}
</style>