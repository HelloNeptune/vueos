<template>
  <div
    id="app"
    v-on:mousemove="appMousemove($event)"
    v-on:mousedown="appMousedown($event)"
    v-on:mouseup="appMouseup($event)"
    :style="uiVariables()"
  >
    <!-- Taskbar -->
    <div ref="taskbar" class="taskbar">
      <div class="vue">
        <svg
          version="1.1"
          viewBox="0 0 261.76 226.69"
          xmlns="http://www.w3.org/2000/svg"
        >
          <g transform="matrix(1.3333 0 0 -1.3333 -76.311 313.34)">
            <g transform="translate(178.06 235.01)">
              <path
                d="m0 0-22.669-39.264-22.669 39.264h-75.491l98.16-170.02 98.16 170.02z"
                fill="#41b883"
              />
            </g>
            <g transform="translate(178.06 235.01)">
              <path
                d="m0 0-22.669-39.264-22.669 39.264h-36.227l58.896-102.01 58.896 102.01z"
                fill="#34495e"
              />
            </g>
          </g>
        </svg>
      </div>
    </div>

    <!-- Windows -->
    <div class="windows">
      <div
        v-for="(window, idx) in windows"
        v-bind:key="idx"
        ref="window"
        class="window"
        :class="{
          active: idx === activeWindow.idx,
          minimized: window.minimized
        }"
        :style="getWindowStyles(idx)"
        v-on:mousedown="appWindowMouseDown(idx)"
      >
        <div class="controls">
          <span class="title">Window {{ idx + 1 }}</span>
          <div class="buttons">
            <span class="button fullscreen"></span>
            <span
              ref="windowMinimize"
              class="button minimize"
              @click="handleWindowMinimize(idx)"
            ></span>
            <span class="button close"></span>
          </div>
        </div>
        <div class="content">
          <component :is="window.component" />
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import Vue from 'vue/dist/vue.esm.js';

const MOUSE_STATES = {
  UP: "up",
  DOWN: "down"
};

Vue.use({
  install(Vue) {
    Vue.prototype.helpers = {
      http: {
        /**
         * @desc
         * @param {String} url -
         */
        get(url) {
          return fetch(url).then((res) => res.json());
        }
      }
    };
  }
});

/**
 * @Component: Static Article
 */
const ArticleComponent = Vue.component("ArticleComponent", {
  template: '<div v-html="content"></div>',
  props: {
    text: {
      type: String,
      default: null
    }
  },
  data() {
    return {
      content: ""
    };
  },
  mounted() {
    this.helpers.http
      .get("https://www.randomtext.me/api/lorem/p-8/20-83")
      .then((res) => (this.content = res.text_out));
  }
});

export default {
  /**
   * @Vue - Components
   * -
   */
  compponents: {
    ArticleComponent
  },

  /**
   * @Vue - Data
   * -
   */
  data() {
    return {
      windows: [],
      activeWindow: null,
      mouseState: MOUSE_STATES.UP,
      ui: {
        taskbar: {
          bounds: { x: null, y: null, w: null, h: null }
        }
      }
    };
  },

  /**
   * @Vue - Computed
   * -
   */
  computed: {},

  /**
   * @Vue - Mounted
   * -
   */
  mounted() {
    this.globalEvents();
    
    this.createWindow("Vue Explorer", ArticleComponent, 20, 20);
    this.createWindow("Vue Explorer", ArticleComponent, 40, 40);
    
    this.$nextTick(() => {
      this.setUiParameters();
    });
  },

  /**
   * @Vue - Updated
   * -
   */
  updated() {},

  /**
   * @Vue - Methods
   * -
   */
  methods: {
    globalEvents() {
      /**
       *
       * [ Native Event: Resize --> window ]
       */
      window.addEventListener("resize", this.windowResized);
    },

    /**
     * @Event
     * @desc Window Resize
     */
    windowResized() {
      this.$nextTick(() => {
        this.setUiParameters();
      });
    },

    /**
     * @Event
     * @desc
     */
    appMousemove: function (e) {
      if (this.mouseState == MOUSE_STATES.DOWN) {
        this.activeWindow.offset.top = e.pageY - this.activeWindow.drag.y;
        this.activeWindow.offset.left = e.pageX - this.activeWindow.drag.x;
      }
    },

    /**
     * @Event
     * @desc
     */
    appMousedown(e) {
      if (e.target && e.target.classList.contains("controls")) {
        this.mouseState = MOUSE_STATES.DOWN;
        this.activeWindow.drag.x = e.pageX - this.activeWindow.offset.left;
        this.activeWindow.drag.y = e.pageY - this.activeWindow.offset.top;
      }
    },

    /**
     * @Event
     * @desc
     */
    appMouseup() {
      this.mouseState = MOUSE_STATES.UP;
      this.activeWindow.offset.top < -20 && (this.activeWindow.offset.top = 0);
    },

    /**
     * @Event
     * @desc
     */
    appWindowMouseDown(idx) {
      this.activeWindow = this.windows[idx];
    },

    /**
     * @desc
     */
    setUiParameters() {
      const rectTaskbar = this.$refs.taskbar.getBoundingClientRect();

      this.ui.taskbar.bounds = {
        x: `${rectTaskbar.x}px`,
        y: `${rectTaskbar.y}px`,
        w: `${rectTaskbar.width}px`,
        h: `${rectTaskbar.height}px`
      };
    },

    /**
     * @desc Recursive css variable string builder from
     * nested objects list
     * @param obj - object list
     * @param prefix - prefix
     */
    cssVarsStringGenerator(obj, prefix) {
      let str = "";

      Object.keys(obj).forEach((key, i) => {
        const value = Object.values(obj)[i];

        typeof value === "object" && value
          ? (str += this.cssVarsStringGenerator(
              value,
              prefix ? `${prefix}-${key}` : key
            ))
          : (str += prefix
              ? (value && `--${prefix}-${key}:${value};`) || ""
              : (value && `--${key}:${value};`) || "");
      });

      return str;
    },

    /**
     * @desc Generates one string css
     * variables from object list
     */
    uiVariables() {
      let variables = {
        taskbar: {
          x: this.ui.taskbar.bounds.x,
          y: this.ui.taskbar.bounds.y,
          w: this.ui.taskbar.bounds.w,
          h: this.ui.taskbar.bounds.h
        }
      };

      return this.cssVarsStringGenerator(variables);
    },

    /**
     * @desc Creates new window
     */
    createWindow(title, component, x, y) {
      const window = {
        title,
        minimized: false,
        component: component,
        drag: { x: 0, y: 0 },
        offset: { top: y, left: x }
      };

      const idx = this.windows.push(window) - 1;

      this.windows[idx].idx = idx;
      this.activeWindow = this.windows[idx];
    },

    /**
     * @desc
     */
    setActiveWindow(idx) {
      this.activeWindow = this.windows[idx];
    },

    /**
     * @desc Generates xy transform value for window
     * position
     */
    getWindowStyles(idx) {
      return {
        transform: `translate(${this.windows[idx].offset.left}px, ${this.windows[idx].offset.top}px)`,
        zIndex: idx === this.activeWindow.idx ? this.windows.length : idx
      };
    },

    /**
     * @desc
     */
    handleWindowMinimize() {
      this.activeWindow.minimized = true;
    }
  }
};
</script>

<style lang="scss">
@import url("https://fonts.googleapis.com/css2?family=Roboto:ital,wght@0,100;0,300;0,400;0,900;1,500&display=swap");

/**
    * Sass Constants
    */
$z: 8888;

/**
    * Sass Mixins
    */
@mixin scrollbar() {
  &::-webkit-scrollbar {
    width: 4px;
    height: 4px;
  }
  &::-webkit-scrollbar-thumb {
    background: transparentize(#96ff60, 0.8);
    border-radius: 15px;
    cursor: pointer;
  }
  &::-webkit-scrollbar-thumb:hover {
    background: #96ff60;
  }
  &::-webkit-scrollbar-track {
    background: rgba(255, 255, 255, 0);
    border-radius: 15px;
  }
}

:root {
  --bg: url("https://4kwallpapers.com/images/wallpapers/lake-mountains-rocks-twilight-sunset-starry-sky-purple-sky-1920x1080-3768.jpg");
  --primary-bg-color: rgba(16, 18, 27, 0.7);
  --primary-shadow: 0 20px 40px -10px rgba(0, 0, 0, 0.75);
  --window-radius: 18px;
}

* {
  box-sizing: border-box;
}

body {
  display: flex;
  justify-content: center;
  background: transparent var(--bg) no-repeat center center / cover;
  min-height: 100vh;
  margin: 0;
  padding: 0;
  font-family: "Roboto", sans-serif;
  overflow: hidden;
}

#app {
  position: relative;
  width: 100%;
  padding: 20px;
}

/* ------ --------- ----------
    * Window Component 
    * ------ --------- --------
    */
.windows {
  position: fixed;
  top: 0;
  left: 0;
  z-index: $z;
}

/* ------ --------- ----------
    * Window Component 
    * ------ --------- --------
    */
.window {
  position: fixed;
  background-color: var(--primary-bg-color);
  border-radius: var(--window-radius);
  box-shadow: var(--primary-shadow);
  @supports (backdrop-filter: blur(20px)) {
    backdrop-filter: blur(20px);
  }
  @supports not (backdrop-filter: blur(20px)) {
    background-color: rgba(16, 18, 27, 0.95);
  }
}

/* ------ --------- ----------
    * Window Component: Controls 
    * ------ --------- --------
    */
.window .controls {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 14px 10px;
  cursor: move;
  height: 45px;
  .title {
    white-space: nowrap;
    padding: 0 10px;
    color: #ffffff;
    pointer-events: none;
    user-select: none;
  }
  .buttons {
    display: flex;
    justify-content: flex-end;
  }
  .buttons .button {
    display: block;
    flex: 0 0 15px;
    max-width: 15px;
    height: 15px;
    border-radius: 5px;
    margin-right: 8px;
    cursor: pointer;
  }
  .close {
    background-color: #f96057;
  }
  .minimize {
    background-color: #f8ce52;
  }
  .fullscreen {
    background-color: #5fcf65;
  }
}

/* ------ --------- ----------
    * Window Component: Content 
    * ------ --------- --------
    */
.window .content {
  width: 450px;
  height: 450px;
  min-width: 200px;
  min-height: 100px;
  resize: both;
  overflow: auto;
  padding: 0 20px 20px 20px;
  opacity: 0.7;
  transition: opacity 0.2s ease;
  @include scrollbar();
  @at-root {
    .window.active .content {
      opacity: 1;
    }
  }
}

/* ------ --------- ----------
    * Window Component: Content -> p
    * ------ --------- --------
    */
.window .content {
  p {
    color: rgba(255, 255, 255, 0.85);
    font-weight: 300;
    font-size: 14px;
    line-height: 22px;
    margin: 0;
  }
  * + p {
    margin-top: 12px;
  }
}

.window.minimized {
  transform: translateY(var(--taskbar-y)) !important;
}

/* ------ --------- ----------
    * Taskbar Component 
    * ------ --------- --------
    */
.taskbar {
  display: flex;
  justify-content: flex-end;
}

/* ------ --------- ----------
    * Taskbar Component: Vue
    * ------ --------- --------
    */
.taskbar .vue {
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 60px;
  height: 60px;
  padding: 10px;
  background-color: rgb(33, 16, 63);
  box-shadow: var(--primary-shadow);
  border-radius: 50%;
  cursor: pointer;
  transition: box-shadow 0.3s ease;
  svg {
    position: relative;
    top: 3px;
    width: 100%;
    height: 100%;
    transition: transform 0.2s ease;
  }
}

.taskbar .vue:hover {
  svg {
    transform: scale(0.9);
  }
  &:before {
    transform: scaleX(1);
    opacity: 1;
  }
}

.taskbar .vue:before {
  content: "";
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  opacity: 0;
  z-index: -1;
  border-radius: 50%;
  pointer-events: none;
  transform: scaleX(0);
  background-color: rgb(33, 16, 63);
  transition: transform 0.2s ease, opacity 0.1s ease;
  animation: vue-hover 1.2s linear alternate infinite;
}

$vue-anim-primary-color: rgba(255, 255, 255, 0);
@keyframes vue-hover {
  0% {
    box-shadow: 0 -20px 30px -10px transparentize($vue-anim-primary-color, 0%);
    transform: scaleY(1.1);
  }
  25% {
    box-shadow: 20px 0 30px 0 transparentize($vue-anim-primary-color, 0%);
    transform: scaleX(1.1);
  }
  50% {
    box-shadow: 0 10px 10px 0 transparentize($vue-anim-primary-color, 0%);
    transform: scaleX(0.9);
  }
  75% {
    box-shadow: -20px -10px 10px -10px
      transparentize($vue-anim-primary-color, 0%);
    transform: scale(1.1);
  }
  100% {
    box-shadow: 0 -20px 30px -10px transparentize($vue-anim-primary-color, 0%);
    transform: scaleY(1.1);
  }
}
</style>
