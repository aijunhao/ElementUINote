<template>
  <div
    :class="carouselClasses"
    @mouseenter.stop="handleMouseEnter"
    @mouseleave.stop="handleMouseLeave">

    <!-- 切换按钮及轮播内容，轮播图主体内容 -->
    <div
      class="el-carousel__container"
      :style="{ height: height }">

      <!-- 
        左右切换箭头按钮
        1. 添加了动画
        2. 按钮中 v-show 设置显示时机。
        3. @mouseenter 是鼠标进入按钮事件 handleButtonEnter()
        4. @mouseleave 是鼠标离开按钮事件 handleButtonLeave()
        5. @click.stop 绑定事件，阻止事件冒泡 throttledArrowClick()
       -->
      <transition
        v-if="arrowDisplay"
        name="carousel-arrow-left">
        <button
          type="button"
          v-show="(arrow === 'always' || hover) && (loop || activeIndex > 0)"
          @mouseenter="handleButtonEnter('left')"
          @mouseleave="handleButtonLeave"
          @click.stop="throttledArrowClick(activeIndex - 1)"
          class="el-carousel__arrow el-carousel__arrow--left">
          <i class="el-icon-arrow-left"></i>
        </button>
      </transition>
      <transition
        v-if="arrowDisplay"
        name="carousel-arrow-right">
        <button
          type="button"
          v-show="(arrow === 'always' || hover) && (loop || activeIndex < items.length - 1)"
          @mouseenter="handleButtonEnter('right')"
          @mouseleave="handleButtonLeave"
          @click.stop="throttledArrowClick(activeIndex + 1)"
          class="el-carousel__arrow el-carousel__arrow--right">
          <i class="el-icon-arrow-right"></i>
        </button>
      </transition>

      <!-- carousel-item 子组件插槽 -->
      <slot></slot>
    </div>

    <!-- 
      指示器
      1. indicatorPosition 指示器位置
      2. @mouseenter 是鼠标进入事件 throttledIndicatorHover() 节流方法
    -->
    <ul
      v-if="indicatorPosition !== 'none'"
      :class="indicatorsClasses">
      <li
        v-for="(item, index) in items"
        :key="index"
        :class="[
          'el-carousel__indicator',
          'el-carousel__indicator--' + direction,
          { 'is-active': index === activeIndex }]"
        @mouseenter="throttledIndicatorHover(index)"
        @click.stop="handleIndicatorClick(index)">
        <button class="el-carousel__button">
          <span v-if="hasLabel">{{ item.label }}</span>
        </button>
      </li>
    </ul>
  </div>
</template>

<script>
// 第三方节流防抖包
import throttle from 'throttle-debounce/throttle';
import { addResizeListener, removeResizeListener } from 'element-ui/src/utils/resize-event';

export default {
  name: 'ElCarousel',

  props: {
    // 初始状态激活的幻灯片的索引，从 0 开始
    initialIndex: {
      type: Number,
      default: 0
    },
    height: String,
    // 指示器的触发方式
    trigger: {
      type: String,
      default: 'hover'
    },
    autoplay: {
      type: Boolean,
      default: true
    },
    // 自动切换的时间间隔
    interval: {
      type: Number,
      default: 3000
    },
    // 指示器的位置
    indicatorPosition: String,
    // ？？？？
    indicator: {
      type: Boolean,
      default: true
    },
    // 切换箭头的显示时机
    arrow: {
      type: String,
      default: 'hover'
    },
    // 走马灯的类型，轮播图和卡片
    type: String,
    loop: {
      type: Boolean,
      default: true
    },
    // 走马灯展示的方向
    direction: {
      type: String,
      default: 'horizontal',
      validator(val) {
        return ['horizontal', 'vertical'].indexOf(val) !== -1;
      }
    }
  },

  data() {
    return {
      // carousel-item 的 dom 节点
      items: [],
      // 激活的 index
      activeIndex: -1,
      containerWidth: 0,
      // 定时器对象
      timer: null,
      // 是否为 hover 状态
      hover: false
    };
  },

  computed: {
    /**
     * 箭头按钮是否显示
     * arrow 为 never 或者 direction 为 vertical 垂直时不显示
     */
    arrowDisplay() {
      return this.arrow !== 'never' && this.direction !== 'vertical';
    },

    hasLabel() {
      return this.items.some(item => item.label.toString().length > 0);
    },

    /**
     * carousel 全局类
     * el-carousel: 父类
     * el-carousel--[horizontal/vertical]: 水平/垂直样式类
     * el-carousel--card: 卡片样式类
     */
    carouselClasses() {
      const classes = ['el-carousel', 'el-carousel--' + this.direction];
      if (this.type === 'card') {
        classes.push('el-carousel--card');
      }
      return classes;
    },

    /**
     * 指示器样式类
     * el-carousel__indicators: 父类
     * el-carousel__indicators--[horizontal/vertical]: 水平/垂直样式类
     * el-carousel__indicators--labels：？？？
     * el-carousel__indicators--outside: type 为 card 卡片或者 indicatorPosition 为 outside 外侧时样式
     */
    indicatorsClasses() {
      const classes = ['el-carousel__indicators', 'el-carousel__indicators--' + this.direction];
      if (this.hasLabel) {
        classes.push('el-carousel__indicators--labels');
      }
      if (this.indicatorPosition === 'outside' || this.type === 'card') {
        classes.push('el-carousel__indicators--outside');
      }
      return classes;
    }
  },

  watch: {
    items(val) {
      if (val.length > 0) this.setActiveItem(this.initialIndex);
    },

    // 激活的 index 索引
    activeIndex(val, oldVal) {
      this.resetItemPosition(oldVal);
      // 发送change事件
      if (oldVal > -1) {
        this.$emit('change', val, oldVal);
      }
    },

    // 自动播放
    autoplay(val) {
      val ? this.startTimer() : this.pauseTimer();
    },

    // 是否循环
    loop() {
      this.setActiveItem(this.activeIndex);
    }
  },

  methods: {
    /**
     * 鼠标进入 Carousel 组件
     */
    handleMouseEnter() {
      this.hover = true;
      this.pauseTimer();
    },
    /**
     * 鼠标离开 Carousel 组件
     */
    handleMouseLeave() {
      this.hover = false;
      this.startTimer();
    },

    /**
     * 项目是否在场景内
     * item: 子项
     * index: 索引
     * return：'left'/'right'/false
     */
    itemInStage(item, index) {
      const length = this.items.length;
      // index === length-1 当前为最后一项
      // item.inStage 当前项目在场景内
      // items[0].active 第一个项目激活状态
      // items[index + 1] 当前项目后有至少一个项目
      // items[index + 1].active 当前项目后一个项目处于激活状态
      // 判断结果为左移，return 'left'
      if (index === length - 1 && item.inStage && this.items[0].active ||
        (item.inStage && this.items[index + 1] && this.items[index + 1].active)) {
        return 'left';
      } else if (index === 0 && item.inStage && this.items[length - 1].active ||
        (item.inStage && this.items[index - 1] && this.items[index - 1].active)) {
        return 'right';
      }
      return false;
    },

    /**
     * 鼠标进入箭头按钮
     * arraw: 'left' / 'right'
     */
    handleButtonEnter(arrow) {
      // 垂直无效
      if (this.direction === 'vertical') return;
      this.items.forEach((item, index) => {
        if (arrow === this.itemInStage(item, index)) {
          item.hover = true;
        }
      });
    },

    /**
     * 鼠标离开箭头按钮
     */
    handleButtonLeave() {
      // 垂直无效
      if (this.direction === 'vertical') return;
      this.items.forEach(item => {
        item.hover = false;
      });
    },

    /**
     * 更新子组件节点
     * 子组件的 name 为 ElCarouselItem，筛选出所有符合要求的子组件，即
     */
    updateItems() {
      this.items = this.$children.filter(child => child.$options.name === 'ElCarouselItem');
    },

    /**
     * 重置 item 子项位置
     */
    resetItemPosition(oldIndex) {
      this.items.forEach((item, index) => {
        // 子组件方法
        item.translateItem(index, this.activeIndex, oldIndex);
      });
    },

    // 开始轮播，只是控制索引
    playSlides() {
      // 如果 当前激活索引 < (数组长度-1)，索引自加翻页。
      // 如果超出了，并且 loop 为 true 循环，则索引置 0
      if (this.activeIndex < this.items.length - 1) {
        this.activeIndex++;
      } else if (this.loop) {
        this.activeIndex = 0;
      }
    },

    // 暂停轮播，清除定时器
    pauseTimer() {
      if (this.timer) {
        clearInterval(this.timer);
        this.timer = null;
      }
    },

    // 开始轮播
    startTimer() {
      // (interval 间隔时间为负数，autoplay 为 false，timer 定时器不为空) 时返回
      if (this.interval <= 0 || !this.autoplay || this.timer) return;
      this.timer = setInterval(this.playSlides, this.interval);
    },

    /**
     * 设置当前页
     * index: 索引
     */
    setActiveItem(index) {
      // 字符串处理，如果索引是字符串，说明是指定名字的
      if (typeof index === 'string') {
        const filteredItems = this.items.filter(item => item.name === index);
        if (filteredItems.length > 0) {
          index = this.items.indexOf(filteredItems[0]);
        }
      }
      index = Number(index);
      // 非正常数字和浮点数处理
      if (isNaN(index) || index !== Math.floor(index)) {
        console.warn('[Element Warn][Carousel]index must be an integer.');
        return;
      }
      let length = this.items.length;
      const oldIndex = this.activeIndex;
      if (index < 0) {
        // 右移尽头处理
        this.activeIndex = this.loop ? length - 1 : 0;
      } else if (index >= length) {
        // 左移尽头处理
        this.activeIndex = this.loop ? 0 : length - 1;
      } else {
        // 普通情况
        this.activeIndex = index;
      }
      if (oldIndex === this.activeIndex) {
        this.resetItemPosition(oldIndex);
      }
    },

    prev() {
      this.setActiveItem(this.activeIndex - 1);
    },

    next() {
      this.setActiveItem(this.activeIndex + 1);
    },

    /**
     * 指示器点击事件
     */
    handleIndicatorClick(index) {
      this.activeIndex = index;
    },

    /**
     * 指示器鼠标悬浮事件
     */
    handleIndicatorHover(index) {
      if (this.trigger === 'hover' && index !== this.activeIndex) {
        this.activeIndex = index;
      }
    }
  },

  created() {
    // 箭头点击的回调函数，使用 throttle 节流, 300ms 内只调用一次
    this.throttledArrowClick = throttle(300, true, index => {
      this.setActiveItem(index);
    });
    // 鼠标悬浮指示器的回调函数，使用 throttle 节流，300ms 内只调用一次
    this.throttledIndicatorHover = throttle(300, index => {
      this.handleIndicatorHover(index);
    });
  },

  mounted() {
    console.log(this);
    this.updateItems();
    this.$nextTick(() => {
      // 添加尺寸更新监听
      addResizeListener(this.$el, this.resetItemPosition);
      // 初始化索引
      if (this.initialIndex < this.items.length && this.initialIndex >= 0) {
        this.activeIndex = this.initialIndex;
      }
      this.startTimer();
    });
  },

  beforeDestroy() {
    // 移除尺寸更新监听
    if (this.$el) removeResizeListener(this.$el, this.resetItemPosition);
    this.pauseTimer();
  }
};
</script>
