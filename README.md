背景：Ant Design Vue 官网中有 Calendar 日历组件，但提供的示例不支持多选，禁选功能

自定义封装组件

### Calendar/index.vue:

```
<template>
  <div class="calendar">
    <div class="select" :style="{ width }">
      <a-select
        v-model="curYear"
        placeholder="请选择"
        :options="yearOptions"
        style="width: 100px; margin-right: 10px"
      />
      <a-select
        v-model="curMonth"
        placeholder="请选择"
        :options="monthOptions"
        style="width: 80px; margin-right: 10px"
      />
      <a-button type="primary" @click="handleQuickChange('prev')">上一月</a-button>
      <a-button type="primary" @click="handleQuickChange('next')">下一月</a-button>
    </div>

    <table class="calendar-table" :style="{ width }">
      <thead>
        <tr>
          <th v-for="(item, i) in weeks" :key="i">{{ item }}</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(dates, i) in res" :key="i" :style="{ height: tbodyHeight }">
          <td
            v-for="(item, index) in dates"
            :key="index"
            :class="{
              notCurMonth: !item.isCurMonth,
              currentDay: item.date === curDate,
              oldDay: oldSelectable ? undefined : item.date < curDate,
              selectDay: item.isSelected,
              rangeSelectd: item.isRangeSelected,
            }"
            @click="handleItemClick(item, i, index)"
            @mouseover="handleItemMove(item, i, index)"
          >
            <span>{{ item.date }}</span>
            <slot :data="item" />
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script>
import { getDaysInMonth, handleCrateDate, handleCreateDatePicker, parseTime } from './index';
export default {
  name: 'Calendar',
  props: {
    selectMode: {
      type: String,
      default: 'click', // click: 点击，mouseover滑动
    },
    startOfWeek: {
      type: Number,
      default: 1, //表头默认开始为周一
    },
    selectable: {
      type: Boolean,
      default: true, // 是否可选择
    },
    oldSelectable: {
      type: Boolean,
      default: false, // 过去的时间是否可选择
    },
    selectedValues: {
      type: Array, // 回显的日期集合
      default: () => [],
    },
    width: {
      type: String,
      default: '70%',
    },
    tbodyHeight: {
      type: String,
      default: '60px',
    },
  },
  data() {
    return {
      monthOptions: [],
      yearOptions: [],
      weeks: ['周一', '周二', '周三', '周四', '周五', '周六', '周日'],
      curYear: new Date().getFullYear(), // 当前年
      curMonth: new Date().getMonth(), // 当前月
      days: 0, // 当前月总共天数
      curDate: parseTime(new Date().getTime()), // 当前日期 yyyy-MM-dd 格式，用来匹配是否是当前日期
      prevDays: [], // 非当前月的上一月展示的日期
      rearDays: [], // 非当前月的下一月展示的日期
      curDays: [], // 当前月的日期
      showDays: [], // 总共展示的42个日期
      res: [], // 二维数组
      selectedDates: [], // 选中的日期
      selectedMode: true, // true表示点击， false表示滑动
      moveIndex: [], // 两个，第一个是起始，第二个是结束
      canMove: true, // 当moveIndex数组有一个值时，可以触发滑动
    };
  },
  computed: {},
  watch: {
    curMonth: {
      handler(val) {
        this.handleGetDays(this.curYear, val, this.startOfWeek);
      },
    },
    curYear: {
      handler(val) {
        this.handleGetDays(val, this.curMonth, this.startOfWeek);
      },
    },
  },
  created() {
    this.weeks.unshift(...this.weeks.splice(this.startOfWeek - 1));
    this.handleGetDays(this.curYear, this.curMonth, this.startOfWeek);
    this.selectedMode = this.selectMode === 'click';
  },
  mounted() {
    this.monthOptions = handleCreateDatePicker().months;
    this.yearOptions = handleCreateDatePicker().years;
    this.selectedDates = this.selectedValues;
  },
  methods: {
    handleGetDays(year, month, startOfWeek) {
      this.showDays = [];
      // 通过年月获取对应的天数
      this.days = getDaysInMonth(year, month);
      // 通过年月获取当月1号的星期
      let firstDayOfWeek = new Date(`${year}-${month + 1}-01`).getDay();

      // 处理周起始日
      const obj = {
        1: this.weeks[0],
        2: this.weeks[1],
        3: this.weeks[2],
        4: this.weeks[3],
        5: this.weeks[4],
        6: this.weeks[5],
        0: this.weeks[6],
      };
      const firstDayInCN = obj[firstDayOfWeek];

      // 获取当前星期到星期一的索引
      const index = this.weeks.indexOf(firstDayInCN);

      if (firstDayOfWeek === 0) {
        // 星期天为0 星期一为1 ，以此类推
        firstDayOfWeek = 7;
      }

      this.prevDays = handleCrateDate(year, month, 1, index + 1, 'prev', this.selectedValues);
      this.rearDays = handleCrateDate(year, month, 1, 42 - this.days - index, 'rear', this.selectedValues);
      this.curDays = handleCrateDate(year, month, 1, this.days, undefined, this.selectedValues);
      this.showDays.unshift(...this.prevDays);
      this.showDays.push(...this.curDays);
      this.showDays.push(...this.rearDays);
      this.res = this.handleFormatDates(this.showDays);
    },
    handleFormatDates(arr, size = 7) {
      // 传入长度42的原数组，最终转换成二维数组
      const arr2 = [];
      for (let i = 0; i < size; i++) {
        const temp = arr.slice(i * size, i * size + size);
        arr2.push(temp);
      }
      return arr2;
    },
    handleTableHead(start) {
      const sliceDates = this.weeks.splice(start - 1);
      this.weeks.unshift(...sliceDates);
    },
    handleItemClick(item, i, j) {
      if (!this.selectable) return;
      if (!this.oldSelectable && item.date < this.curDate) {
        return;
      } else {
        if (this.selectedMode) {
          this.$nextTick(() => {
            this.res[i][j].isSelected = !this.res[i][j].isSelected;
            if (this.res[i][j].isSelected) {
              this.selectedDates.push(this.res[i][j].date);
              this.selectedDates = Array.from(new Set(this.selectedDates));
            } else {
              this.selectedDates.splice(this.selectedDates.indexOf(item.date), 1);
            }
            this.$emit('dateSelected', this.selectedDates);
          });
        } else {
          // 滑动模式下，第一次点击是起始，第二次点击是结束
          const index = i * 7 + j;
          this.canMove = true;
          if (this.moveIndex.length === 1) {
            this.canMove = false;
          }
          if (this.moveIndex.length === 2) {
            this.showDays.forEach((item) => {
              item.isSelected = false;
              item.isRangeSelected = false;
            });
            this.canMove = true;
            this.moveIndex.length = 0;
          }
          this.moveIndex.push(index);
          this.moveIndex.sort((a, b) => a - b);
          this.selectedDates = this.showDays.slice(this.moveIndex[0], this.moveIndex[1] + 1);
          this.selectedDates.length !== 0 && this.$emit('dateSelected', this.selectedDates);
        }
      }
    },
    handleItemMove(data, i, j) {
      if (this.canMove && !this.selectedMode) {
        const index = i * 7 + j;
        this.showDays.forEach((item) => {
          item.isSelected = false;
          item.isRangeSelected = false;
        });
        // 让第一个日期和最后一个日期显示蓝色高亮
        this.showDays[index].isSelected = true;
        this.showDays[this.moveIndex[0]].isSelected = true;

        // 不同情况的判断，当用户的鼠标滑动进日期的索引小于起始日期的索引，要做if else处理
        if (this.moveIndex[0] < index) {
          for (let i = this.moveIndex[0] + 1; i < index; i++) {
            this.showDays[i].isRangeSelected = true;
          }
        } else {
          for (let i = index + 1; i < this.moveIndex[0]; i++) {
            this.showDays[i].isRangeSelected = true;
          }
        }
      }
    },
    handleQuickChange(type) {
      if (type === 'prev') {
        this.curMonth--;
        if (this.curMonth === -1) {
          this.curMonth = 11;
          this.curYear -= 1;
        }
      } else if (type === 'next') {
        this.curMonth++;
        if (this.curMonth === 12) {
          this.curMonth = 0;
          this.curYear += 1;
        }
      }
    },
  },
};
</script>

<style scoped lang="less">
.calendar {
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: column;
}
.select {
  display: flex;
  width: 100%;
  justify-content: end;
}
.calendar-table {
  table-layout: fixed;
  border-collapse: collapse;
  transition: 0.3s;
  thead tr {
    height: 50px;
  }
  tbody tr {
    &:first-child td {
      border-top: 1px solid #238dde;
    }
    td {
      cursor: pointer;
      border-right: 1px solid #238dde;
      border-bottom: 1px solid #238dde;
      &:first-child {
        border-left: 1px solid #238dde;
      }
    }
  }
}

.notCurMonth {
  color: #c0c4cc;
}
.currentDay {
  //   color: #fff;
  //   background-color: #409eff;
}
.oldDay {
  color: #606266;
  background-color: #c0c4cc;
}
.selectDay {
  color: #fff;
  background-color: #238dde;
}
.rangeSelectd {
  color: #fff;
  background-color: #238dde;
}
</style>


```

### Calendar/index.js

组件涉及到的方法：

```
// 获取该月的天数
export const getDaysInMonth = (year, month) => {
  const day = new Date(year, month + 1, 0);
  return day.getDate();
};

// 创建日期 yyyy-MM-dd 格式， 用于创建非当前月的日期
export const handleCrateDate = (year, month, start, end, type, selectedValues) => {
  const arr = [];
  if (type === 'prev') {
    // 上一月
    if (start === end) return [];
    const daysInLastMonth = getDaysInMonth(year, month - 1); // 获取上一个月有多少天
    console.log(`当前月是${month + 1}月, 上一月${month}月的天数是${daysInLastMonth}天`);
    for (let i = daysInLastMonth - end + 2; i <= daysInLastMonth; i++) {
      arr.push({
        date: parseTime(new Date(year, month - 1, i)),
        isCurMonth: false,
        isSelected: selectedValues.includes(parseTime(new Date(year, month - 1, i))),
        isRangeSelected: false,
      });
    }
  } else if (type === 'rear') {
    // 下一月
    for (let i = start; i <= end; i++) {
      arr.push({
        date: parseTime(new Date(year, month + 1, i)),
        isCurMonth: false,
        isSelected: selectedValues.includes(parseTime(new Date(year, month + 1, i))),
        isRangeSelected: false,
      });
    }
  } else {
    // 本月
    for (let i = start; i <= end; i++) {
      arr.push({
        date: parseTime(new Date(year, month, i)),
        isCurMonth: true,
        isSelected: selectedValues.includes(parseTime(new Date(year, month, i))),
        isRangeSelected: false,
      });
    }
  }
  return arr;
};

export const handleCreateDatePicker = () => {
  const years = [];
  const months = [];
  for (let i = 1970; i <= 2099; i++) {
    years.push({
      label: `${i}年`,
      value: i,
    });
  }
  for (let i = 0; i <= 11; i++) {
    months.push({
      label: `${i + 1}月`,
      value: i,
    });
  }
  return {
    years,
    months,
  };
};

/**
 * Parse the time to string
 * @param {(Object|string|number)} time
 * @param {string} cFormat
 * @returns {string | null}
 */
export function parseTime(time, cFormat) {
  if (arguments.length === 0 || !time) {
    return null;
  }
  const format = cFormat || '{y}-{m}-{d}';
  let date;
  if (typeof time === 'object') {
    date = time;
  } else {
    if (typeof time === 'string') {
      if (/^[0-9]+$/.test(time)) {
        // support "1548221490638"
        time = parseInt(time);
      } else {
        // support safari
        // https://stackoverflow.com/questions/4310953/invalid-date-in-safari
        time = time.replace(new RegExp(/-/gm), '/');
      }
    }

    if (typeof time === 'number' && time.toString().length === 10) {
      time = time * 1000;
    }
    date = new Date(time);
  }
  const formatObj = {
    y: date.getFullYear(),
    m: date.getMonth() + 1,
    d: date.getDate(),
    h: date.getHours(),
    i: date.getMinutes(),
    s: date.getSeconds(),
    a: date.getDay(),
  };
  const time_str = format.replace(/{([ymdhisa])+}/g, (result, key) => {
    const value = formatObj[key];
    // Note: getDay() returns 0 on Sunday
    if (key === 'a') {
      return ['日', '一', '二', '三', '四', '五', '六'][value];
    }
    return value.toString().padStart(2, '0');
  });
  return time_str;
}

```

### 引入组件：

```
<Calendar :selectedValues="selectedValues“ @dateSelected="dateSelected"/>

方法：
dateSelected(selectedValues) {
 this.selectedValues=selectedValues;
}

```

效果：

![](https://pic.imgdb.cn/item/664a0bc9d9c307b7e97f082a.gif)

多选：
![](https://pic.imgdb.cn/item/664a0bc9d9c307b7e97f0870.gif)

滑动选择：
![](https://pic.imgdb.cn/item/664a0bcad9c307b7e97f08be.gif)
