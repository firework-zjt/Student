<template>
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
        <button @click="exportAsImage" class="export-btn">导出图片</button>
      </div>
      
      <div ref="scheduleGrid" class="schedule-grid">
        <!-- Header row -->
        <div class="grid-row header-row">
          <div class="grid-cell header-cell"></div>
          <div 
            v-for="(day, index) in days" 
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
            v-for="(day, colIndex) in days" 
            :key="'cell-' + rowIndex + '-' + colIndex" 
            class="grid-cell"
            @click="addCourse(rowIndex, colIndex)"
            @dragover.prevent
            @drop="onDrop($event, rowIndex, colIndex)"
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
  </div>
</template>

<script>
/**
 * Schedule 课程表  点击右侧的课表内容空白区域，然后点击左侧的课程表，就可以显示在表格中了
   * @module Schedule 课程表
   * @author Fireworks
   * @property {Array} subjects 左侧列表的课程数组
   * @property {Number} rows 课程表的行数
   * @property {Number} columns 课程表的列数
   * @property {String} watermarkText 水印文字
   * @event course-added 课程添加事件
   * @event export-complete 导出完成事件
   * @example
 */

import html2canvas from 'html2canvas';
export default {
  name: 'CourseSchedule',
  props: {
    subjects: {
      type: Array,
      default: () => ['数学', '英语', '语文']
    },
    rows: {
      type: Number,
      default: 5
    },
    columns: {
      type: Number,
      default: 5
    },
    watermarkText: {
      type: String,
      default: 'Fireworks的课程表组件'
    }
  },
  data() {
    return {
      selectedSubject: '',
      days: ['周一', '周二', '周三', '周四', '周五'],
      times: [],
      schedule: []
    };
  },
  created() {
    if (this.subjects.length > 0) {
      this.selectedSubject = this.subjects[0];
    }
    this.generateTimeSlots();
    this.initializeSchedule();
  },
  methods: {
    /**
     * @function generateTimeSlots
     * @returns {void
   */
    generateTimeSlots() {
      this.times = [];
      for (let i = 1; i <= this.rows; i++) {
        this.times.push(`第${i}节`);
      }
    },
    /**
     * @function initializeSchedule
     * @returns {void}
    */
    initializeSchedule() {
      this.schedule = [];
      for (let i = 0; i < this.rows; i++) {
        this.schedule[i] = new Array(this.columns).fill('');
      }
    },
    /**
     * @function addCourse
     * @param {Number} row - 行索引
     * @param {Number} col - 列索引
     * @returns {void}
    */
    addCourse(row, col) {
      if (this.selectedSubject) {
        this.$set(this.schedule[row], col, this.selectedSubject);
      }
    },
  /**
   * @function onDragStart
   * @param {Object} event - 事件对象
   * @param {String} subject - 课程名称
   * @returns {void}
    */
    onDragStart(event, subject) {
      event.dataTransfer.setData('text/plain', subject);
      event.dataTransfer.effectAllowed = 'copy';
    },
    /**
    * @function onDrop
    * @param {Object} event - 事件对象
    * @param {Number} row - 行索引
    * @param {Number} col - 列索引
    */
    onDrop(event, row, col) {
      const subject = event.dataTransfer.getData('text/plain');
      if (subject) {
        // Overwrite existing content if any
        this.$set(this.schedule[row], col, subject);
        this.$emit('course-added', { row, col, subject });
      }
    },
   /**
    * function 导出图片
    * @function exportAsImage
    * @returns {Promise<void>}
    */
    async exportAsImage() {
      try {
      this.$refs.watermark.style.display = 'block';
      
      const element = this.$refs.scheduleGrid;
      const canvas = await html2canvas(element, {
        backgroundColor: '#ffffff',
        scale: 2,
        logging: false
      });
      this.$refs.watermark.style.display = 'none';
      const image = canvas.toDataURL('image/png');
      const link = document.createElement('a');
      link.download = '课程表.png';
      link.href = image;
      link.click();
      
      this.$emit('export-complete', image);
    } catch (error) {
      console.error('导出图片失败:', error);
    }
    }
  }
}
</script>

<style scoped>
.course-schedule-container {
  display: flex;
  font-family: Arial, sans-serif;
  border: 1px solid #e0e0e0;
  border-radius: 4px;
  overflow: hidden;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

/* Left side styles */
.subject-list {
  width: 200px;
  background-color: #f5f5f5;
  padding: 15px;
  border-right: 1px solid #e0e0e0;
}

.subject-list h3 {
  margin-top: 0;
  margin-bottom: 15px;
  color: #333;
}

.datalist {
  list-style: none;
  padding: 0;
  margin: 0;
}

.datalist li {
  padding: 10px;
  margin-bottom: 5px;
  background-color: #fff;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.2s;
  user-select: none; /* Prevent text selection during drag */
}

.datalist li:hover {
  background-color: #f0f0f0;
}

.datalist li.active {
  background-color: #e6f7ff;
  border-left: 3px solid #1890ff;
}

/* Drag styles */
.datalist li[draggable=true] {
  cursor: grab;
}

.datalist li[draggable=true]:active {
  cursor: grabbing;
}

/* Right side styles */
.schedule-container {
  flex: 1;
  padding: 15px;
  background-color: #fff;
}

.schedule-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
}

.schedule-header h3 {
  margin: 0;
  color: #333;
}

.export-btn {
  background-color: #1890ff;
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  transition: background-color 0.3s;
}

.export-btn:hover {
  background-color: #40a9ff;
}

/* Grid styles */
.schedule-grid {
  position: relative;
  border: 1px solid #e8e8e8;
  border-radius: 4px;
  overflow: hidden;
}

.grid-row {
  display: flex;
}

.grid-cell {
  flex: 1;
  min-height: 80px;
  border: 1px solid #e8e8e8;
  padding: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
  text-align: center;
}

.header-row {
  background-color: #f5f5f5;
}

.header-cell {
  min-height: 50px;
  font-weight: bold;
}

.time-cell {
  background-color: #f5f5f5;
  min-width: 80px;
  font-weight: bold;
}

.course-item {
  width: 100%;
  height: 100%;
  background-color: #e6f7ff;
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 5px;
}

/* Watermark styles */
.watermark {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%) rotate(-45deg);
  font-size: 48px;
  color: rgba(0, 0, 0, 0.1);
  pointer-events: none;
  white-space: nowrap;
  user-select: none;
}

/* Drop target highlight */
.grid-cell.drag-over {
  background-color: #e6f7ff;
  border: 2px dashed #1890ff;
}

@media (max-width: 768px) {
  .course-schedule-container {
    flex-direction: column;
  }
  
  .subject-list {
    width: 100%;
    border-right: none;
    border-bottom: 1px solid #e0e0e0;
  }
}
</style>