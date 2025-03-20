<template>
  <!-- 模板部分保持不变 -->
  <div class="course-schedule-container">
    <!-- Left side: Subject list -->
    <div class="subject-list">
      <h3>课程列表</h3>
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
    <div class="schedule-container">
      <div class="schedule-header">
        <h3>课程表</h3>
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
          <div class="grid-cell time-cell">{{ time }}</div>
          <div 
            v-for="(day, colIndex) in dayslist" 
            :key="'cell-' + rowIndex + '-' + colIndex" 
            class="grid-cell"
            :class="{ 'drag-over': isDragOver(rowIndex, colIndex) }"
            @dragenter.prevent="onDragEnter(rowIndex, colIndex)"
            @dragleave.prevent="onDragLeave(rowIndex, colIndex)"
            @dragover.prevent="onDragOver($event)"
            @drop="onDrop($event, rowIndex, colIndex)"
            draggable="true"
            @dragstart="onCellDragStart($event, rowIndex, colIndex)"
            @dragend="onDragEnd($event, rowIndex, colIndex)"
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
 * 课程表组件
 * 
 * @component
 * @name CourseSchedule
 * @auther Fireworks
 * @prop {Array} subjects - 课程列表
 * @prop {Array} times - 时间表
 * @prop {Array} dayslist - 星期列表
 * @prop {string} watermarkText - 水印文本
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
      default: () => ['数学', '英语', '语文']
    },
    /**
     * 时间表
     * @type {Array}
     * @default ['1', '2', '3', '4', '5', '6', '7', '8', '9']
     */
    times: {
      type: Array,
      default: () => ['第一节', '第二节', '第三节', '第四节', '第五节', '第六节', '第七节', '第八节', '第九节']
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
     * 开始拖拽课程
     * @param {Event} event - 拖拽事件
     * @param {string} subject - 课程名称
     */
    onDragStart(event, subject) {
      event.dataTransfer.setData('text/plain', subject);
      event.dataTransfer.effectAllowed = 'move';
      const dragImage = document.createElement('div');
      dragImage.textContent = subject;
      dragImage.style.backgroundColor = '#ebf8ff';
      dragImage.style.padding = '10px';
      dragImage.style.borderRadius = '6px';
      dragImage.style.color = '#2b6cb0';
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
            watermark.style.color = 'rgba(66, 153, 225, 0.3)';
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
    // 移除：监听 dragend 事件
  },
  beforeUnmount() {
    // 移除：移除 dragend 事件监听
  }
}
</script>

<style scoped>
/* 样式保持不变 */
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
  width: 220px;
  background-color: #f7faff;
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
  background-color: #ebf4ff;
  transform: translateY(-1px);
}

.datalist li.active {
  background-color: #ebf8ff;
  border-left: 3px solid #4299e1;
  color: #2b6cb0;
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
  background-color: #4299e1;
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
  min-height: 90px;
  border: 1px solid #e6f0ff;
  padding: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  text-align: center;
  transition: background-color 0.2s ease;
}

.header-row {
  background-color: #f0f7ff;
}

.header-cell {
  min-height: 50px;
  font-weight: 600;
  color: #2c5282;
  letter-spacing: 0.5px;
}

.time-cell {
  background-color: #f0f7ff;
  min-width: 80px;
  font-weight: 600;
  color: #2c5282;
}

.course-item {
  width: 100%;
  height: 100%;
  background-color: #ebf8ff;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 8px;
  color: #2b6cb0;
  font-weight: 500;
  box-shadow: 0 2px 5px rgba(66, 153, 225, 0.1);
  transition: all 0.2s ease;
}

.course-item:hover {
  background-color: #bee3f8;
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
  color: rgba(66, 153, 225, 0.08);
  pointer-events: none;
  white-space: nowrap;
  user-select: none;
  transform: rotate(-30deg);
  z-index: 10;
}

.grid-cell.drag-over {
  background-color: #ebf8ff;
  border: 2px dashed #4299e1;
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
  background-color: #4299e1;
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