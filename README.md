# CourseSchedule 组件使用文档

## 概述
`CourseSchedule` 是一个基于 Vue 的交互式课程表组件，支持课程拖拽、导出课程表为图片、清空课程表等功能。适用于课程管理、日程安排等场景。

---

## 安装与引入

### 安装依赖
确保项目中已安装 `html2canvas`，若未安装，可通过以下命令安装：

```bash
npm install html2canvas
```

```
<script>
import CourseSchedule from 'course-schedule'; // 替换为实际路径
export default {
 components: {
    CourseSchedule
  },
  data() {
    return {
      subjects: ['数学', '英语', '语文'],
      times: ['第一节', '第二节', '第三节', '第四节', '第五节', '第六节'],
      dayslist: ['周一', '周二', '周三', '周四', '周五'],
      watermarkText: 'Fireworks的课程表组件',
      mainTitle: '课程表',
      SubjectTitle: '课程列表',
      primaryColor: '#ff4100',
      subjectListWidth: '220px',
      scheduleContainerWidth: 'auto',
      cellWidth: 'auto',
      cellHeight: '90px'
    };
  },
  methods: {
    onCourseAdded(payload) {
      console.log('课程添加成功', payload);
    },
    onCourseRemoved(payload) {
      console.log('课程移除成功', payload);
    },
    onExportComplete(image) {
      console.log('课程表导出完成', image);
    },
    onClearComplete() {
      console.log('课程表已清空');
    },
    onMoveSuccess() {
      console.log('课程移动成功');
    },
    onConfirmRemove(payload) {
      console.log('确认移除课程', payload);
    },
    onCancelRemove() {
      console.log('取消移除课程');
    },
    onDragStart(event, subject) {
      console.log('开始拖拽课程', subject);
    },
    onCellDragStart(event, row, col) {
      console.log('开始拖拽单元格', row, col);
    }
  }
};
</script>

<template>
  <div>
    <CourseSchedule
      :subjects="subjects"
      :times="times"
      :dayslist="dayslist"
      :watermarkText="watermarkText"
      :mainTitle="mainTitle"
      :SubjectTitle="SubjectTitle"
      :primaryColor="primaryColor"
      :subjectListWidth="subjectListWidth"
      :scheduleContainerWidth="scheduleContainerWidth"
      :cellWidth="cellWidth"
      :cellHeight="cellHeight"
      @course-added="onCourseAdded"
      @course-removed="onCourseRemoved"
      @export-complete="onExportComplete"
      @clear-complete="onClearComplete"
      @move-success="onMoveSuccess"
      @confirm-remove="onConfirmRemove"
      @cancel-remove="onCancelRemove"
      @onDragStart="onDragStart"
      @onCellDragStart="onCellDragStart"
    />
  </div>
</template>

```

## 组件属性说明

| 属性名 | 类型 | 默认值 | 说明 | 
| --- | --- | --- | --- | 
| subjects | Array | ['数学', '英语', '语文'] | 课程列表 | 
| times | Array | ['第一节', '第二节', ..., '第六节'] | 时间表 |
| dayslist | Array | ['周一', '周二', ..., '周五'] | 星期列表 | 
| watermarkText | String | 'Fireworks的课程表组件' | 水印文本 | 
| mainTitle | String | '课程表' | 主标题 | 
| SubjectTitle | String | '课程列表' | 课程列表标题 |
| primaryColor | String | '#ff4100' | 主体颜色 | 
| subjectListWidth | String | '220px' | 左侧课程列表宽度 |
| scheduleContainerWidth | String | 'auto' | 右侧课程表容器宽度 |
| cellWidth | String | 'auto' | 右侧课程表内单元格宽度 |
| cellHeight | String | '90px' | 右侧课程表内单元格高度 |

---

## 组件事件说明
| 事件名 | 说明 |
| --- | --- | 
| course-added | 课程添加成功时触发，返回添加课程的信息（{ row, col, subject }） | 
| course-removed | 课程移除成功时触发，返回移除课程的信息（{ row, col }） |
| export-complete | 课程表导出完成时触发，返回导出的图片数据（image） |
| clear-complete | 课程表清空完成时触发 |
| move-success | 课程移动成功时触发 | 
| confirm-remove | 确认移除课程时触发，返回移除课程的信息（{ row, col }） |
| cancel-remove | 取消移除课程时触发 | 
| onDragStart | 开始拖拽课程时触发，返回拖拽事件和课程名称（event, subject） |
| onCellDragStart | 开始拖拽单元格时触发，返回拖拽事件和单元格的行、列索引（event, row, col） |

---


## 注意事项
1、导出图片：使用 exportAsImage 方法导出课程表为图片时，确保页面中没有跨域资源，否则可能导致导出失败。

2、颜色样式：组件中的颜色样式使用了 CSS 变量，可通过修改 primaryColor 属性来改变主体颜色。

3、动态调整：组件支持动态调整课程表大小和布局，可通过 subjectListWidth、scheduleContainerWidth 等属性自定义宽度和高度。

---

## 许可证
MIT License

---

## 贡献
欢迎提交 Issue 或 Pull Request 以改进组件功能。

---

## 作者
Fireworks

## 版本
1.0.5