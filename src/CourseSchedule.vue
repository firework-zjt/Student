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
  font-family: 'Segoe UI', 'Microsoft YaHei', sans-serif;
  border: none;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
  background-color: #ffffff;
}

/* Left side styles */
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

.export-btn {
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

.export-btn:hover {
  background-color: #3182ce;
  transform: translateY(-1px);
  box-shadow: 0 4px 6px rgba(66, 153, 225, 0.25);
}

.export-btn:active {
  transform: translateY(0);
}

/* Grid styles */
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

/* Watermark styles - keeping the original positioning but updating style */
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

/* Drop target highlight */
.grid-cell.drag-over {
  background-color: #ebf8ff;
  border: 2px dashed #4299e1;
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