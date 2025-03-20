<template>
  <div class="course-schedule-container">
    <!-- Left side: Subject list -->
    <div class="subject-list">
      <h3>课程列表</h3>
      <ul class="datalist">
        <li v-for="(subject, index) in subjects" :key="index" :class="{ active: selectedSubject === subject }"
          @click="selectedSubject = subject" draggable="true" @dragstart="onDragStart($event, subject)">
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
          <div v-for="(day, index) in days" :key="'day-' + index" class="grid-cell header-cell">
            {{ day }}
          </div>
        </div>

        <!-- Time slots rows -->
        <div v-for="(time, rowIndex) in times" :key="'time-' + rowIndex" class="grid-row">
          <div class="grid-cell time-cell">{{ time }}</div>
          <div v-for="(day, colIndex) in days" :key="'cell-' + rowIndex + '-' + colIndex" class="grid-cell"
            :class="{ 'drag-over': isDragOver(rowIndex, colIndex) }"
            @dragenter.prevent="onDragEnter(rowIndex, colIndex)" @dragleave.prevent="onDragLeave(rowIndex, colIndex)"
            @dragover.prevent="onDragOver($event)" @drop="onDrop($event, rowIndex, colIndex)">
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
import { defineComponent } from 'vue';
import html2canvas from 'html2canvas';

export default defineComponent({
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
  emits: ['course-added', 'export-complete'],
  data() {
    return {
      selectedSubject: '',
      days: ['周一', '周二', '周三', '周四', '周五'],
      times: [],
      schedule: [],
      dragTarget: null
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
    generateTimeSlots() {
      this.times = [];
      for (let i = 1; i <= this.rows; i++) {
        this.times.push(`第${i}节`);
      }
    },
    initializeSchedule() {
      this.schedule = [];
      for (let i = 0; i < this.rows; i++) {
        this.schedule[i] = new Array(this.columns).fill('');
      }
    },
    addCourse(row, col) {
      if (this.selectedSubject) {
        this.$set(this.schedule[row], col, this.selectedSubject);
        this.$emit('course-added', { row, col, subject: this.selectedSubject });
      }
    },
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
    onDragEnter(row, col) {
      this.dragTarget = { row, col };
    },
    onDragLeave(row, col) {
      if (this.dragTarget && this.dragTarget.row === row && this.dragTarget.col === col) {
        this.dragTarget = null;
      }
    },
    onDragOver(event) {
      event.preventDefault();
      event.dataTransfer.dropEffect = 'move';
    },
    isDragOver(row, col) {
      return this.dragTarget && this.dragTarget.row === row && this.dragTarget.col === col;
    },
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
            this.$set(this.schedule[row], col, subject);
            this.$emit('course-added', { row, col, subject });
            this.dragTarget = null;
          }, 300);
        }, 10);
      }
    },
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
    }
  }
});
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