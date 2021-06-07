<template>
  <div
    id="app"
    v-on:mousemove="handleMouseMove($event)"
    v-on:mousedown="handleMouseDown($event)"
    v-on:mouseup="handleMouseUp($event)"
    :style="uiVariables()"
  >
    <!-- Taskbar -->
    <div ref="taskbar" class="taskbar">
      <div class="vue" ref="vue" @click="handleVueClick()">
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
        v-bind:key="window.id"
        ref="window"
        class="window"
        :v-id="window.id"
        :style="windowUiVariables(idx)"
        :class="{
          active: activeItem && activeItem.id === window.id,
          maximized: window.maximized,
          minimized: minimizedWindows.find((mwId) => mwId === window.id),
          'user-interacting':
            window.id === activeItem.id && window.userInteracting
        }"
        v-on:mousedown="handleWindowMouseDown(window.id)"
      >
        <div class="window-controls">
          <span class="title">{{ window.title }}</span>
          <div class="buttons">
            <span
              class="button fullscreen"
              @click="handleWindowMaximize(window.id)"
            ></span>
            <span
              class="button minimize"
              @click="handleWindowMinimize(window.id)"
            ></span>
            <span
              class="button close"
              @click="handleWindowClose(window.id)"
            ></span>
          </div>
        </div>
        <div
          class="content"
          v-if="true && handleWindowContentInit(window.id)"
          v-on:mouseup="handleWindowContentMouseUp(window.id)"
          v-on:mousedown="handleWindowContentMouseDown(window.id)"
        >
          <component
            :is="window.component"
            :window-id="window.id"
            :file="getFile(window.fileId)"
            v-on:on-save="handleAppOnSave"
          />
        </div>
      </div>
    </div>

    <!-- Desktop -->
    <div class="desktop">
      <div
        v-for="(item, idx) in items"
        v-bind:key="item.id"
        ref="item"
        class="desktop-item"
        :class="{
          app: item.type === 'app',
          file: item.type === 'file',
          new: item.new,
          active: activeItem && activeItem.id === item.id,
        }"
        :style="desktopItemUiVariables(idx)"
        v-on:click="handleFileClick(item.id)"
        v-on:dblclick="handleFileDoubleClick(item.id)"
        v-on:mousedown="handleFileMouseDown(item.id)"
      >
        <div class="wrapper">
          <div class="icon">
            <i :class="getIcon(item)" />
          </div>
          <span class="name">
            {{ getName(item) }}
          </span>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import Vue from 'vue/dist/vue.esm.js';

// Dump vuedit file
// #
const aNote = 'JTdCJTIyaWQlMjIlM0ElMjJlZTJmMTllZDIyNDYlMjIlMkMlMjJmaWxlVHlwZSUyMiUzQSUyMnZ1ZWRpdCUyMiUyQyUyMmZpbGVOYW1lJTIyJTNBJTIyYSUyMG5vdGUlMjIlMkMlMjJjb250ZW50JTIyJTNBJTIyJTNDYiUzRUxldCdzJTIwbWFrZSUyMGElMjBzdGF0ZW1lbnQhJTNDJTJGYiUzRSUzQ2RpdiUzRSUzQ2klM0VCdSUyMGJpciUyMGl0YWxpayUyMG1ldGluLiUzQyUyRmklM0UlM0NiciUzRSUzQyUyRmRpdiUzRSUzQ2RpdiUzRSUzQ3UlM0UlQzMlQjZuZW1saSUyMGJpciUyMHV5YXIlQzQlQjEuJTNDJTJGdSUzRSUzQ2JyJTNFJTNDJTJGZGl2JTNFJTIyJTdE';

const parse = JSON.parse;
const stringify = JSON.stringify;

const MOUSE_STATES = {
  UP: "up",
  DOWN: "down"
};

const APP_FILE_TYPES = {
  vuedit: 'vuedit',
  story: 'story'
}

const APP_ITEM_TYPES = {
  window: 'window',
  file: 'file',
  app: 'app'
}

/**
 * @desc Matches & Closest polyfills
 * @jonathantneal https://github.com/jonathantneal/closest/blob/master/src/index.js
 */
const ElementPrototype = window.Element.prototype;
if (typeof ElementPrototype.matches !== 'function') {
  /**
   * @desc Add matches method to html element prototype
   * @param selector -
   */
  ElementPrototype.matchs = ElementPrototype.msMatchesSelector ||  ElementPrototype.mozMatchesSelector || ElementPrototype.webkitMatchesSelector || 
    function matches(selector) {
      let idx = 0;
      let element = this;
      const elements = (element.document || element.ownerDocument).querySelectorAll(selector);

      while (elements[idx] && elements[idx] !== element) {
        idx++;
      }
      return Boolean(elements[idx]);
    }
}

if (typeof ElementPrototype.closest !== 'function') {
  /**
   * @desc Add closest method to html element prototype
   * @param selector -
   */
  ElementPrototype.closest = function closest(selector) {
    let element = this;
    
    while (element && element.nodeType === Node.ELEMENT_NODE) {
      if (element.matches(selector)) {
        return element;
      }
      element = element.parentNode;
    }
    return null;
  }
}

const helpers = {

  /**
   * @Vue - data
   */
  data() {
    return {
      helpers: {
        http: {
          /**
           * @desc Makes an http get request
           * @param url -
           */
          get(url) {
            return fetch(url).then((res) => res.json());
          }
        },

        /**
         * @desc Generates random ID
         */
        randid() {
          return Math.random().toString(16).slice(2, -1);
        }
      }
    };
  }
};

const system = {

  /**
   * @Vue - computed
   */
  computed: {
    _helpers() {
      return helpers.data().helpers
    }
  },

  /**
   * @Vue - Methods
   */
  methods: {
    system(app) {
      const context = this;

      return {
        /**
         * @desc
         * @param data -
         */
        encode(data) {
          return btoa(encodeURIComponent(stringify(data)));
        },

        /**
         * @desc
         * @param data -
         */
        decode(data) {
          return parse(decodeURIComponent(atob(data)));
        },

        /**
         * @desc
         * @param fileType -
         * @param fileName -
         * @param content -
         */
        saveFile(fileType, fileName, content) {
          const id = app.windowId || context._helpers.randid();
          
          localStorage.setItem(`file-${app.windowId}`, this.encode({
            id,
            fileType,
            fileName,
            content
          }));
        },

        /**
         * @desc
         */
        getFiles() {
          const files = [];
          let f = 0;

          do {
            let file = null;
            const key = localStorage.key(f);
            const record = key && key.includes('file-') && localStorage.getItem(key);

            if (record) {
              try {
                file = this.decode(record)
              } catch(e) { console.warn('File cannot load(!)') }

              file && files.push({ type: APP_ITEM_TYPES.file, ...file});
            }

            f++;
            if (!key) f = null;
          } while (f !== null)

          if (files.length === 0) {
            files.push({
              type: APP_ITEM_TYPES.file,
              ...this.decode(aNote)
            })
          }

          return files;
        },

        /**
         * @desc 
         */
        getApps() {
          const apps = Object.keys(Vue.options.components).filter(app => {
            const component = Vue.options.components[app];
            return typeof component === 'function' && component.options.config;
          });

          return apps.map(app => ({
            type: APP_ITEM_TYPES.app,
            appConfig: {
              appName: app,
              ...Vue.options.components[app].options.config
            }
          }));
        }
      }
    }
  }
}

/* <b>Let's make a statement!</b>
<br>
<i>Bu bir italik metin.</i>
<br>
<u>Çok </u> */

const SaveComponentTemplate = `
  <div component="Save" :class="{shown: shown}">
    <div class="form">
      <div class="form-element">
        <span class="label">Filename</span>
        <input v-model="fileName" placeholder="give a name to this" type="text"></input>
      </div>
    </div>
    <button class="button" v-on:click="handleSaveClick">
      Save    
      <i class="gg-loadbar"></i>
    </button>
  </div>
`;
const SaveComponent = Vue.component('SaveComponent', {

  /**
   * @Vue -template
   */
  template: SaveComponentTemplate,

  /**
   * @Vue - mixins
   */
  mixins: [helpers],

  /**
   * @Vue - data
   */
  data() {
    return {
      shown: false,
      fileName: null,
      resolve: null,
      reject: null
    }
  },

  /**
   * @Vue - methods
   */
  methods: {

    /**
     * @Event
     * @desc
     */
    handleSaveClick() {
      if (this.fileName && this.resolve) {
        this.resolve(this.fileName);
        this.setState(false);
      }
    },

    /**
     * @desc
     */
    run(file) {
      if (file) {
        this.fileName = file.fileName;
      }

      return new Promise((resolve, reject) => {
        this.setState(true, resolve, reject);
      });
    },
    
    /**
     * @desc
     * @param status -
     * @param resolve -
     * @param reject -
     */
    setState(status, resolve, reject) {
      if (status) {
        this.resolve = resolve;
        this.reject = reject;
      }
      else {
        this.resolve = null;
        this.reject = null;
      }

      this.disableContent(status);
      this.shown = status;
    },

    /**
     * @desc
     * @param disable -
     */
    disableContent(disable) {
      const appContent = this.$parent.$refs['app-content'];

      if (this.$parent && appContent) {
        if (disable) {
          appContent.style.filter = 'blur(3px)';
          appContent.style.opacity = '0.5';
        }
        else {
          appContent.style.filter = null;
          appContent.style.opacity = null;
        }
      }
    }
  }
});

/**
 * @Component: Static Article
 * @from @souporserious https://codepen.io/souporserious/pen/xBpEj
 */
const EditorComponentTemplate = `
  <div component="Editor">
    <Save ref="saveModal" />
    <div ref="app-content" class="app-content">
      <div class="editor-controls">
        <a href="javascript:void(0);" data-role='bold' v-on:click="execute('bold')">B</a>
        <a href="javascript:void(0);" data-role='italic' v-on:click="execute('italic')">I</a>
        <a href="javascript:void(0);" data-role='underline' v-on:click="execute('underline')">U</a>
        <a href="javascript:void(0);" data-role='justifyleft' v-on:click="execute('justifyleft')"><i class="menu-left"></i></a>
        <a href="javascript:void(0);" data-role='justifycenter' v-on:click="execute('justifycenter')"><i class="menu-center"></i></a>
        <a href="javascript:void(0);" data-role='justifyright' v-on:click="execute('justifyright')"><i class="menu-right"></i></a>
        <a v-on:click="handleSaveClick" class="save">
          Save
          <i class="gg-software-download" />
        </a>
      </div>
      <div ref="content" class="editor-content" contenteditable v-html="content"></div>
    </div>
  </div>
`;
const EditorComponent = Vue.component("EditorComponent", {
  /**
   * @Vueos - config
   */
  config: {
    name: 'Vuedit',
    icon: 'gg-format-text',
    ext: APP_FILE_TYPES.vuedit
  },

  components: { Save: SaveComponent },

  /**
   * @Vue - template
   */
  template: EditorComponentTemplate,

  /**
   * @Vue - mixins
   */
  mixins: [helpers, system],

  /**
   * @Vue - props
   */
  props: {
    windowId: {
      type: String,
      default: null
    },
    file: {
      type: Object,
      default: null
    },
  },

  /**
   * @Vue - created
   */
  created() {
    if (this.file) {
      this.content = this.file.content;
    }
  },

  /**
   * @Vue - data
   */
  data() {
    return {
      content: '',
    };
  },

  /**
   * @Vue - methods
   */
  methods: {

    /**
     * @Event
     * @desc
     */
    handleSaveClick() {
      this.save();
    },

    /**
     * @desc
     * @param filename -
     */
    saveToDisk(fileName) {
      const content = this.$refs.content.innerHTML;
      this.system(this).saveFile(APP_FILE_TYPES.vuedit, fileName, content);
      this.$emit('on-save', this.windowId);
    },

    /**
     * @desc
     * 
     */
    save() {

      if (!this.file) {
        this.$refs.saveModal.run().then(
          fileName => this.saveToDisk(fileName)
        );
      }
      else this.saveToDisk(this.file.fileName);
    },

    /**
     * @desc
     * @param command -
     */
    execute(command) {
      document.execCommand(command, false);
    }
  }
});

/**
 * @Component: Static Article
 * @from @ovens https://codepen.io/ovens/pen/EeprWN
 */
const ArticleComponentTemplate =  `
  <div>
    <div v-bind:style="style.p" v-for="(v, idx) in [0,0,0,0,0,0]" v-html="sentence(idx)">
    </div>
  </div>
`;
const ArticleComponent = Vue.component("ArticleComponent", {
  /**
   * @Vueos - config
   */
  config: {
    name: 'Unreal Story',
    icon: 'gg-file-document',
    ext: APP_FILE_TYPES.story
  },

  /**
   * @Vue - template
   */
  template: ArticleComponentTemplate,

  /**
   * @Vue - mixins
   */
  mixins: [helpers],
  
  /**
   * @Vue - props
   */
  props: {},

  /**
   * @Vue - data
   */
  data() {
    return {
      n: [
        "bird", "clock", "boy",
        "plastic", "duck", "teacher",
        "old lady", "professor", "hamster", "dog"
      ],
      v: [
        "kicked", "ran", "flew",
        "dodged", "sliced", "rolled",
        "died", "breathed", "slept",
        "killed"
      ],
      a: [
        "beautiful", "lazy", "professional",
        "lovely", "dumb", "rough",
        "soft", "hot", "vibrating",
        "slimy"
      ],
      b: [
        "slowly", "elegantly", "precisely",
        "quickly", "sadly", "humbly",
        "proudly", "shockingly", "calmly",
        "passionately"],
      p: [
        "down", "into", "up", "on", "upon",
        "below", "above", "through", "across",
        "towards"
      ],
      content: '',
      style: {
        p: {
          color: '#efefef',
          fontWeight: 300,
          fontSize: '13px',
          marginBottom: '10px',
          lineHeight: '1.8em'
        }
      }
    };
  },

  /**
   * @Vue - methods
   */
  methods: {
    /**
     * @desc 
     * @param max -
     */
    rand(max) {
      return Math.floor(Math.random() * (max)) + 1;
    },

    /**
     * @desc
     * @param total -
     * @param randMax -
     */
    fillRand(total, randMax) {
      return [...Array(total)].map( _ => this.rand(randMax));
    },

    /**
     * @desc
     * @param idx -
     */
    sentence(idx) {
      let text = '';
      const totalWord = this.rand(5);

      for (let s = 0; s <= totalWord; s++) {
        const [r1, r2, r3, r4, r5, r6] = this.fillRand(6, 10);

        text += (idx === 0 ? 'The ' : '') + 
          this.a[r1] + " " + this.n[r2] + " " + 
          this.b[r3] + " " + this.v[r4] + " because some " + 
          this.n[r1] + " " + this.b[r1] + " " + 
          this.v[r1] + " " + this.p[r1] + " a " + 
          this.a[r2] + " " + this.n[r5] + " which, became a " + 
          this.a[r3] + ", " + this.a[r4] + " " + 
          this.n[r6] + "."
      }

      return text;
    }
  }
});

export default {

  /**
   * @Vue - Mixins
   */
  mixins: [helpers, system],

  /**
   * @Vue - Data
   * -
   */
  data() {
    const uiDefaultTaskbar = {
      verticalGap: 20,
      horizontalGap: 20,
      height: 60
    };

    const uiDefaultWindow = {
      minimizedAnimDuration: "0.35s",
      minimizedWidth: 250,
      minimizedHeight: 45,
      minimizedGap: 20
    };

    const uiDefaultFile = {
      width: 100,
      height: 100
    }

    const taskbarBounds = {
      x: null,
      y: null,
      w: null,
      h: null 
    };

    return {
      windows: [],
      files: [],
      apps: [],
      folders: [],
      items: [],
      minimizedWindows: [],
      windowFocusSequence: [],
      activeItem: null,
      mouseState: MOUSE_STATES.UP,
      ui: {
        itemDefaults: {
          cssVariables: {},
          drag: { x: 0, y: 0 },
          offset: { top: 0, left: 0 }
        },
        default: {
          taskbar: uiDefaultTaskbar,
          window: uiDefaultWindow,
          file: uiDefaultFile
        },
        taskbar: {
          bounds: taskbarBounds
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
   * @Vue - Watch
   * -
   */
  watch: {
    /**
     * @Watch
     * @desc update window focues sequence every
     * activeItem change
     */
    activeItem(newValue, oldValue) {
      newValue &&
      newValue.type &&
      newValue.type === APP_ITEM_TYPES.window &&
      this.updateWindowFocusSequence(newValue.id);
    }
  },

  /**
   * @Vue - beforeCreate
   */
  created() {

    // Get vueos apps
    // #
    this.apps = this.system(this).getApps();
    
    // Get files from hdd :)
    // #
    this.files = this.system(this).getFiles();

    this.items = [
      ...this.apps,
      ...this.files
    ];

    this.items.forEach(item => {
      Object.keys(this.ui.itemDefaults).forEach(d => {
        item[d] = this.ui.itemDefaults[d]
      })

      !item.id && (item.id = this.helpers.randid());
    });
  },

  /**
   * @Vue - Mounted
   * -
   */
  mounted() {
    this.globalEvents();

    this.createWindow("Unreal Story", ArticleComponent, 150, 150);

    this.$nextTick(() => {
      this.setUiParameters();
      this.organizeItems();
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
      window.addEventListener("resize", this.handleBrowserWindowResize);
    },

    /**
     * @Event Resize
     * @desc Window Resize
     */
    handleBrowserWindowResize() {
      this.$nextTick(() => {
        this.setUiParameters();
        this.organizeItems();
        this.organizeMinimizedWindows();
      });
    },

    /**
     * @Event MouseMove
     * @desc
     */
    handleMouseMove: function (e) {
      if (
        this.activeItem &&
        this.mouseState == MOUSE_STATES.DOWN && (
        (this.activeItem.type === APP_ITEM_TYPES.window && 
        !this.isWindowMinimized(this.activeItem.id) &&
        !this.isWindowMaximized(this.activeItem.id)) ||
        (this.activeItem.type === APP_ITEM_TYPES.file ||
        this.activeItem.type === APP_ITEM_TYPES.app))
      ) {
        this.activeItem.offset.top = e.pageY - this.activeItem.drag.y;
        this.activeItem.offset.left = e.pageX - this.activeItem.drag.x;
        this.$forceUpdate();
      }
    },

    /**
     * @Event MouseDown
     * @desc
     */
    handleMouseDown(e) {
      if (e.target && (
        e.target.classList.contains("window-controls") ||
        e.target.closest('.desktop-item'))
      ) {
        this.mouseState = MOUSE_STATES.DOWN;
        this.activeItem.drag.x = e.pageX - this.activeItem.offset.left;
        this.activeItem.drag.y = e.pageY - this.activeItem.offset.top;
      }
    },

    /**
     * @Event MouseUp
     * @desc
     */
    handleMouseUp() {
      if (!this.activeItem) return;

      this.mouseState = MOUSE_STATES.UP;
      this.activeItem.offset.top < -20 && (this.activeItem.offset.top = 0);
      this.activeItem.userInteracting = false;
    },

    /**
     * @Event MouseDown
     * @desc
     */
    handleWindowMouseDown(id) {
      const window = this.windows.find((w) => w.id === id);
      this.activeItem = window;
      this.activeItem.userInteracting = true;
    },

    /**
     * @Event MouseDown
     * @desc
     */
    handleWindowContentMouseDown(id) {
      const window = this.windows.find((w) => w.id === id);
      window.userInteracting = true;
    },

    /**
     * @Event MouseUp
     * @desc
     */
    handleWindowContentMouseUp(id) {
      const window = this.windows.find((w) => w.id === id);
      window.userInteracting = false;
    },

    /**
     * @Event Resize
     * @desc
     */
    handleWindowResize(record, window, contentDomRef) {
      window.cssVariables.bounds = {
        width: contentDomRef.offsetWidth,
        height: contentDomRef.offsetHeight
      };

      this.$forceUpdate();
    },

    /**
     * @Event Init
     * @desc
     */
    handleWindowContentInit(id) {
      if (!this.$refs.window) return true;
      const window = this.windows.find((w) => w.id === id);

      if (window && !window.resizeObserver) {
        const domRef = this.$refs.window.find(
          (w) => w.getAttribute("v-id") === id
        );
        const content = domRef && domRef.querySelector(".content");

        if (!domRef || !content) return true;

        /**
         *
         * [Observer: Mutation --> window ]
         */
        window.resizeObserver = new MutationObserver((record) =>
          this.handleWindowResize(record, window, content)
        );

        window.resizeObserver.observe(content, {
          attributes: true,
          attributeFilter: ["style"]
        });

        if (!window.cssVariables.bounds) {
          //this.handleWindowResize(null, window, domRef);
        }
      }

      return true;
    },

    /**
     * @Event MouseDown
     * @desc
     */
    handleFileMouseDown(id) {
      const item = this.items.find((i) => i.id === id);
      this.activeItem = item;
    },

    /**
     * @Event Click
     * @desc
     */
    handleFileClick(id) {
    },

    /**
     * @Event Double Click
     * @desc
     */
    handleFileDoubleClick(id) {
      const item = this.items.find((i) => i.id === id);
      
      if (item.type === APP_ITEM_TYPES.app) {
        this.openApp(item);
      }
      else if (item.type === APP_ITEM_TYPES.file) {
        this.openFile(item);
      }
    },

    /**
     * @Event Click
     * @desc
     */
    handleVueClick() {
    },

    /**
     * @Event On Save
     * @desc
     */
    handleAppOnSave(fileId) {
      const wIdx = this.windows.findIndex(w => w.id === fileId);
      const item = this.items.find(i => i.id === fileId);
      const file = this.system(this).decode(
        localStorage.getItem(`file-${fileId}`)
      );


      if (item) {
        item.content = file.content;
      }
      else {
        Object.keys(this.ui.itemDefaults).forEach(d => {
          file[d] = this.ui.itemDefaults[d]
        })

        file.type = APP_ITEM_TYPES.file;
        file.new = true;

        const idx = this.items.push(file) - 1;

        this.items[idx].offset = this.getDesktopItemPosition(idx);
        this.windows[wIdx].fileId = fileId;
      }
    },

    /**
     * @desc
     * @param file -
     */
    openFile(file) {
      const app = this.apps.find(app => app.appConfig.ext === file.fileType);
      const fileOpened = this.windows.find(w => w.id === file.id); 

      if (app && !fileOpened) {
        this.openApp(app, file);
      }

      if (fileOpened) {
        // focus window
      }
    },

    /**
     * @desc
     * @param app -
     * @param file -
     */
    openApp(app, file) {
      const { name, appName } = app.appConfig;
      const component = Vue.options.components[appName];
      const title = name + (file && (' - ' + file.fileName) || '');
    
      if (component) {
        this.createWindow(
          title,
          component,
          null,
          null,
          file && file.id || null
        );
      }
    },

    /**
     * @desc Creates new window
     */
    createWindow(title, component, x, y, id) {
      const lastFocusedWindowId = this.windowFocusSequence.slice(-1)[0];
      const lastFocusedWindow = this.windows.find(w => w.id === lastFocusedWindowId); 

      // Re-position new window 
      // after last focused window
      if (lastFocusedWindow) {
        !x && (x = lastFocusedWindow.offset.left + 20);
        !y && (y = lastFocusedWindow.offset.top + 20);
      }
      
      const window = {
        id: id || this.helpers.randid(),
        type: APP_ITEM_TYPES.window,
        title,
        component,
        fileId: id,
        minimized: false,
        userInteracting: false,
        drag: { x: 0, y: 0 },
        offset: { 
          top: y || 170, left: x || 170 },
        cssVariables: {},
      };

      const idx = this.windows.push(window) - 1;

      this.windows[idx].idx = idx;
      this.activeItem = this.windows[idx];
      this.updateWindowFocusSequence(window.id);
    },

    /**
     * @desc
     */
    setUiParameters() {
      const rectTaskbar = this.$refs.taskbar.getBoundingClientRect();

      this.ui.taskbar.bounds = {
        x: rectTaskbar.x,
        y: rectTaskbar.y,
        w: rectTaskbar.width,
        h: rectTaskbar.height
      };
    },

    /**
     * @desc Recursive css variable string or list builder from
     * nested objects list
     * @param obj - object list
     * @param prefix - prefix
     */
    cssVarsGenerator(obj, type = "string") {
      /**
       * @desc Example output: '--prop-val: val; --prop-val2: val'
       */
      const stringGenerator = (obj) => {
        const generator = (obj, prefix) => {
          let str = "";
          Object.keys(obj).forEach((key, idx) => {
            const value = Object.values(obj)[idx];

            typeof value === "object" && value
              ? (str += generator(value, prefix ? `${prefix}-${key}` : key))
              : (str += prefix
                  ? (!!value && "") || `--${prefix}-${key}:${value};`
                  : (!!value && "") || `--${key}:${value};`);
          });
          return str;
        };
        return generator(obj);
      };

      /**
       * @desc Example output: {
       *  '--prop-val': val',
       *  '--prop-val2': 'val'
       * }
       */
      const listGenerator = (obj) => {
        const list = {};
        const stringified = stringGenerator(Object.assign({}, obj));

        stringified.split(";").forEach((rule) => {
          if (rule) {
            const [key, value] = rule.split(":");
            list[key] = value;
          }
        });

        return list;
      };

      return type !== "string" ? listGenerator(obj) : stringGenerator(obj);
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
        },
        ...this.ui.default
      };

      return this.cssVarsGenerator(variables);
    },

    /**
     * @desc Generates styles for window
     * @param idx -
     */
    windowUiVariables(idx) {
      const window = this.windows[idx];
      let posx = window.offset.left;
      let posy = window.offset.top;

      return {
        transform: `translate(${posx}px, ${posy}px)`,
        zIndex: this.windowFocusSequence.findIndex((wId) => wId === window.id),
        ...this.cssVarsGenerator(window.cssVariables, "list"),
      };
    },

    /**
     * @desc Generates style for desktop items
     * @param idx -
     */
    desktopItemUiVariables(idx, id) {
      const item = this.items[idx];
      let posx = item.offset.left;
      let posy = item.offset.top;

      return {
        transform: `translate(${posx}px, ${posy}px)`,
        zIndex: this.activeItem && this.activeItem.id === item.id ? 1 : 0,
        ...this.cssVarsGenerator(item.cssVariables, "list"),
      };
    },

    /**
     * @desc
     * @param id -
     */
    isWindowMinimized(id) {
      return this.minimizedWindows.findIndex((wmId) => wmId === id) !== -1;
    },

    /**
     * @desc
     * @param id -
     */
    isWindowMaximized(id) {
      const idx = this.windows.findIndex((w) => w.id === id);
      return this.windows[idx].maximized;
    },

    /**
     * @desc
     * @param Id -
     */
    updateWindowFocusSequence(id) {
      const windowIdx = this.windowFocusSequence.findIndex((wId) => wId === id);

      if (windowIdx === -1) {
        this.windowFocusSequence.push(id);
      } else {
        this.$delete(this.windowFocusSequence, windowIdx);
        this.windowFocusSequence.push(id);
      }
    },

    /**
     * @desc
     * @param id -
     */
    organizeMinimizedWindows() {
      const uiDefaults = this.ui.default.window;

      /**
       * @desc
       * @param parentWidth -
       */
      const calculateWidth = (parentWidth) => {
        const defaultWidth =
          uiDefaults.minimizedWidth + uiDefaults.minimizedGap;
        const minimizedWindows = this.minimizedWindows.length;

        if (defaultWidth * minimizedWindows > parentWidth) {
          return parentWidth / minimizedWindows - uiDefaults.minimizedGap;
        }

        return uiDefaults.minimizedWidth;
      };

      this.minimizedWindows.forEach((mwId, idx) => {
        const windowIdx = this.windows.findIndex((w) => w.id === mwId);
        const rectVue = this.$refs.vue.getBoundingClientRect();
        const boundsTaskbar = this.ui.taskbar.bounds;
        const height = uiDefaults.minimizedHeight;
        const width = calculateWidth(boundsTaskbar.w - rectVue.width);

        const baseX =
          boundsTaskbar.x +
          boundsTaskbar.w -
          rectVue.width -
          width -
          uiDefaults.minimizedGap;

        const x = baseX - uiDefaults.minimizedGap * idx - width * idx;

        const y =
          boundsTaskbar.y +
          boundsTaskbar.h / 2 -
          uiDefaults.minimizedHeight / 2;

        this.windows[windowIdx].cssVariables.minimized = {
          x: `${x}px`,
          y: `${y}px`,
          width: `${width}px`,
          height: `${height}px`
        };
      });
    },
    
    /**
     * @desc
     */
    organizeItems() {
      this.items.map((item, idx) => {
        item.offset = this.getDesktopItemPosition(idx);
        return item;
      });
    },

    /**
     * @desc
     * @param idx -
     */
    getDesktopItemPosition(idx) {
      const grid = this.getDesktopGrid();
      const column = this.items.length > grid.totalItemsInColumn
          ? parseInt((idx / (grid.totalItemsInColumn % this.items.length))) 
          : 0
        ;
      const startTop = grid.top + 20;
      const top = ((idx % (grid.totalItemsInColumn)) * this.ui.default.file.height) + startTop;
      const left = (column * this.ui.default.file.width) + 20;

      return { top, left };
    },

    /**
     * @desc
     */
    getDesktopGrid() {
      const { h: taskbarBoundsH } = this.ui.taskbar.bounds;
      const [wW, wH] = [window.innerWidth, window.innerHeight];
      const top = (this.ui.default.taskbar.verticalGap * 2) + taskbarBoundsH
      const totalItemsInRow = parseInt(wW / this.ui.default.file.width);
      const totalItemsInColumn = parseInt((wH - top) / this.ui.default.file.height);

      return { top, totalItemsInRow, totalItemsInColumn }
    },

    /**
     * @desc
     * @param item -
     */
    getExt(item) {
      if (item.type === APP_ITEM_TYPES.file) {
        return '.' + item.fileType;
      }

      return '';
    },

    /**
     * @desc
     * @param item -
     */
    getName(item) {
      let name = item.type === APP_ITEM_TYPES.app
        ? item.appConfig.name
        : item.fileName + this.getExt(item)
      ;

      return name;
    },
    
    /**
     * @desc
     * @param item -
     */
    getIcon(item) {
      let icon = item.type === APP_ITEM_TYPES.app && item.appConfig.icon || '';

      if (item.type === APP_ITEM_TYPES.file) {
        const app = this.apps.find(app => 
          app.appConfig.ext === item.fileType
        );

        icon = app && app.appConfig.icon || '';
      }

      return icon;
    },

    /**
     * @desc
     * @param fileId -
     */
    getFile(fileId) {
      if (!fileId) return;
      const file = this.items.find(file => file.id === fileId);

      if (file) {
        return file;
      }
    },

    /**
     * @desc
     */
    setActiveWindow(idx) {
      this.activeItem = this.windows[idx];
    },

    /**
     * @desc
     * @param id -
     */
    toggleWindowMinimize(id) {
      const idx = this.windows.findIndex((w) => w.id === id);
      const minimizedIdx = this.minimizedWindows.findIndex(
        (wmId) => wmId === id
      );

      if (this.windows[idx].maximized) {
        this.toggleWindowMaximize(id);
      }

      if (minimizedIdx !== -1) {
        this.$delete(this.minimizedWindows, minimizedIdx);
      }
      else {
        this.minimizedWindows.push(id);
      }

      this.organizeMinimizedWindows();
    },

    /**
     * @desc
     * @param id -
     */
    toggleWindowMaximize(id) {
      const idx = this.windows.findIndex((w) => w.id === id);
      const isMinimized = this.isWindowMinimized(this.activeItem.id);

      if (isMinimized) {
        this.handleWindowMinimize(id);
      }

      this.windows[idx].maximized = !this.windows[idx].maximized;
      this.$forceUpdate();
    },

    /**
     * @desc
     * @param id -
     */
    handleWindowMinimize(id) {
      this.toggleWindowMinimize(id);
    },

    /**
     * @desc
     * @param idx -
     */
    handleWindowMaximize(id) {
      this.toggleWindowMaximize(id);
    },

    /**
     * @desc
     * @param id -
     */
    handleWindowClose(id) {
      const idx = this.windows.findIndex((w) => w.id === id);
      const minimizedIdx = this.minimizedWindows.findIndex(
        (mwId) => mwId === id
      );
      const focusSequenceIdx = this.windowFocusSequence.findIndex(
        (wfsId) => wfsId === id
      );

      this.$delete(this.windows, idx);

      if (focusSequenceIdx !== -1) {
        this.$delete(this.windowFocusSequence, focusSequenceIdx);
      }

      if (minimizedIdx !== -1) {
        this.$delete(this.minimizedWindows, minimizedIdx);
        this.organizeMinimizedWindows();
      }

      if (!this.windows.length) {
        this.activeItem = null;
        return;
      }

      // Make last focused window active
      // #
      if (!!this.windows.length && this.activeItem.id === id) {
        const lastFocusedWindow = this.windowFocusSequence.slice(-1).pop();
        const availableWindows = this.windows.filter(
          (w) => !this.isWindowMinimized(w.id)
        );

        // If all windows are minimized
        // set last focused window active
        if (!availableWindows.length) {
          this.setActiveWindow(
            this.windows.findIndex((w) => w.id === lastFocusedWindow)
          );
          return;
        }

        // Otherwise set last focused
        // window from not minimized ones
        const targetWindow = this.windowFocusSequence
          .filter((wId) => availableWindows.find((aW) => aW.id === wId))
          .slice(-1)
          .pop();
        
        this.setActiveWindow(
          this.windows.findIndex((w) => w.id === targetWindow )
        );
      }
    }
  }
};
</script>

<style lang="scss">
@import url('https://fonts.googleapis.com/css2?family=Roboto:ital,wght@0,100;0,300;0,400;0,900;1,500&display=swap');
@import url('https://unpkg.com/css.gg/icons/all.css');
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
  --primary-text-shadow: 0 0 3px rgb(0 0 0 / 75%);
  --window-radius: 18px;
  --vueos-transition-fn: cubic-bezier(0.19, 1, 0.22, 1);
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
  transition: transform 0.5s var(--vueos-transition-fn),
    width 0.35s var(--vueos-transition-fn) !important;
  transform-origin: 0% 100%;
  width: calc((var(--bounds-width) * 1px));
  height: calc((var(--bounds-height) * 1px) + 45px);
  overflow: hidden;
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
.window .window-controls {
  position: relative;
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
    width: calc(100% - 70px);
    text-overflow: ellipsis;
    overflow: hidden;
  }
  .buttons {
    position: absolute;
    top: 50%;
    right: 0;
    display: flex;
    justify-content: flex-end;
    width: 70px;
    transform: translateY(-50%);
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
  overflow: hidden;
  width: var(--minimized-width);
  height: var(--minimized-height);
  transform: translate(var(--minimized-x), var(--minimized-y)) !important;
}

.window.maximized {
  $edge-gap: 10px;
  $top: calc((var(--taskbar-verticalGap) * 1px * 2) + (var(--taskbar-height) * 1px));
  $left: $edge-gap;
  width: calc(100vw - #{$edge-gap * 2});
  height: calc(100vh - (#{$edge-gap * 2} + (calc((var(--taskbar-verticalGap) * 1px * 2) + (var(--taskbar-height) * 1px)))));
  transform: translateY($top) translateX($left) !important;
  z-index: 999999;
  .content {
    width: 100% !important;
    height: calc(100% - 50px) !important;
    resize: none;
  }
}

.window.user-interacting {
  transition: none !important;
}

/* ------ --------- ----------
    * Taskbar Component 
    * ------ --------- --------
    */
.taskbar {
  position: absolute;
  top: calc(var(--taskbar-verticalGap) * 1px);
  left: calc(var(--taskbar-horizontalGap) *1px);
  width: calc(100% - (var(--taskbar-horizontalGap) * 1px) * 2);
  height: calc(var(--taskbar-height) * 1px);
  display: flex;
  justify-content: flex-end;
  z-index: $z - 1;
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

/* ------ --------- ----------
    * Desktop Component
    * ------ --------- --------
    */
.desktop {
  position: fixed;
  top: 0;
  left: 0;
  z-index: $z - 2;
}

.desktop-item {
  position: fixed;
  user-select: none;
  cursor: pointer;
  width: 90px;
  height: 90px;
  &:hover {
    &:before {
      opacity: 1;
    }
  }
}

.desktop-item .wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
}

/* ------ --------- ----------
    * Desktop Component: Desktop Item
    * ------ --------- --------
    */
.desktop-item:before {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: 100%;
  height: 100%;
  padding: 10px;
  opacity: 0;
  transform: translate(-50%, -55%);
  background-color: rgba(255, 255, 255, 0.12);
  transition: all 0.3s var(--vueos-transition-fn);
  border-radius: 10px;
  backdrop-filter: blur(10px);
  z-index: -1;
}

.desktop-item .icon {
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  text-align: center;
  padding: 6px;
  font-weight: 600;
  font-size: 12px;
  line-height: 1em;
  text-transform: capitalize;
  border: 1px solid rgba(255, 255, 255, 0.1);
  background: #fff;
  border-radius: 8px;
  min-width: 50px;
  min-height: 50px;
  box-sizing: border-box;
  color: #3aa776;
  box-shadow: var(--primary-shadow);
  backdrop-filter: blur(10px);
}

.desktop-item .name {
  font-weight: 300;
  color: #fff;
  margin-top: 8px;
  width: 100%;
  text-overflow: ellipsis;
  overflow: hidden;
  text-align: center;
  white-space: nowrap;
  text-shadow: var(--primary-text-shadow);
}

.desktop-item.app {
  .icon {
    background-color: #311b50;
    color: #fff;
    &:before {
      content: 'app';
      display: block;
      position: absolute;
      bottom: 2px;
      left: 100%;
      background-color: #41b883;
      padding: 2px 3px;
      border-radius: 6px;
      margin-left: -12px;
    }
  }
}

.desktop-item.active {
  .name {
    overflow: visible;
    word-break: break-all;
    white-space: normal;
  }
}

.desktop-item.new {
  @keyframes newFile {
    0% {
      transform: scale(0);
    }
    100% {
      transform: scale(1);
    }
  }
  .wrapper {
    animation-delay: 0.1s;
    animation: newFile 0.3s var(--vueos-transition-fn);
  }
}

/* ------ --------- ----------
    * Component: Editor
    * ------ --------- --------
    */
[component="Editor"] {
  $controls-color: #ADB5B9;
  $border-color: rgba(255, 255, 255, 0.1);
  margin: 0 -10px;
  height: 100%;
  .editor-controls {
    display: flex;
    align-items: center;
    width: 100%;
    height: 35px;
    border: 1px solid $border-color;
    border-bottom: none;
    border-radius: calc(var(--window-radius) / 2) calc(var(--window-radius) / 2) 0 0;
    background: #292535;
    white-space: nowrap;
    overflow: hidden;
    a {
      display: inline-block;
      width: 35px;
      height: 35px;
      vertical-align: top;
      line-height: 38px;
      text-decoration: none;
      text-align: center;
      cursor: pointer;
      color: $controls-color;
      transition: color 0.3s var(--vueos-transition-fn);
      &:hover {
        color: #ffffff;
        [class^="menu"] {
          background: #ffffff;
        }
      }
    }
    [data-role="bold"] {
      font-weight: bold;
    }
    [data-role="italic"] {
      font-style: italic;
    }
    [data-role="underline"] {
      text-decoration: underline;
    }
  }

  [class^="menu"] {
    position: relative;
    top: 48%;
    display: block;
    width: 65%;
    height: 2px;
    margin: 0 auto;
    background: $controls-color;
    transition: background 0.3s var(--vueos-transition-fn);
    &:before {
      @extend [class^="menu"];
      content: '';
      top: -5px;
      width: 80%;
    }
    &:after {
      @extend [class^="menu"];
      content: '';
      top: 3px;
      width: 80%;
    }
  }

  .menu-left {
    &:before, &:after {
      margin-right: 4px;
    }
  }
  .menu-right {
    &:before, &:after {
      margin-left: 4px;
    }
  }

  .save {
    display: flex !important;
    align-items: center;
    width: auto !important;
    margin-left: auto;
    margin-right: 10px;
    i {
      margin-left: 10px;
    }
  }
  
  .editor-content {
    max-width: 100%;
    min-width: 100%;
    min-height: calc( 100% - 35px );
    padding: 12px;
    overflow: auto;
    font-family: Helvetica, sans-serif;
    font-size: 12px;
    border: 1px solid $border-color;
    border-top: none;
    border-radius: 0 0 var(--window-radius) var(--window-radius);
    background: #332e42;
    outline: none;
    color: #fff;
    font-size: 14px;
    @include scrollbar();
  }

  .app-content {
    height: 100%;
  }
}

button.button {
  position: relative;
  display: flex;
  border: none;
  box-shadow: none;
  background-color: #57f9b0;
  border-radius: 20px;
  flex-direction: column;
  align-items: center;
  color: #060606;
  font-weight: 600;
  font-size: 12px;
  width: 100px;
  height: 30px;
  justify-content: center;
  transition: box-shadow 0.3s var(--vueos-transition-fn), transform 0.3s var(--vueos-transition-fn);
  box-shadow: 0 10px 8px -7px rgb(34 125 84 / 0%);
  cursor: pointer;
  cursor: pointer;
  i {
    position: absolute;
    bottom: 10px;
    left: 50%;
    transition: all 0.3s var(--vueos-transition-fn);
    transform: translateY(15px) translateX(100%);
    opacity: 0;
    color: #57f9b0;
    user-select: none;
  }
  &:hover {
    //box-shadow: 0 10px 8px -7px #227d54;
    i {
      transform: translateY(15px) translateX(-50%);
      opacity: 1;
    }
  }
}

.form-element {
  display: flex;
  align-items: center;
  border-radius: 4px;
  width: 100%;
  height: 30px;
}

.form-element:hover {
  input[type="text"] {
    background-color: transparentize(#fff, 0.1);
  }
}

.form-element .label {
  display: flex;
  align-items: center;
  color: #e8e8e8;
  font-size: 12px;
  border: 2px solid #ffffff;
  height: 100%;
  padding-right: 10px;
  padding-top: 5px;
  padding-bottom: 5px;
  padding-left: 10px;
  background: #352256;
  border-radius: 8px 0 0 8px;
}

input[type="text"] {
  background-color: #ffffff;
  font-size: 12px;
  border: none;
  outline: none;
  -webkit-tap-highlight-color: transparent;
  width: 100%;
  height: 100%;
  padding: 0 10px;
  transition: background-color 0.3s var(--vueos-transition-fn);
  border-left: none;
  border-radius: 0 8px 8px 0;
  &:focus {
    background-color: transparentize(#fff, 0.1);
  }
}

/* ------ --------- ----------
    * Component: Save
    * ------ --------- --------
    */
[component="Save"] {
  position: absolute;
  display: flex;
  align-items: center;
  justify-content: center;
  visibility: hidden;
  opacity: 0;
  top: 45px;
  left: 50%;
  width: calc(100% - 20px);
  height: 50px;
  z-index: 999;
  transform: translateX(-50%) translateY(-40px);
  transition: all 0.3s var(--vueos-transition-fn);
  border-radius: 16px;
  padding: 10px;
  .form {
    display: flex;
    align-items: center;
    width: 100%;
  }
  input {
    border-radius: 0;
  }
  button {
    border-radius: 0 8px 8px 0;    
  }
  &.shown {
    opacity: 1;
    visibility: visible;
    transform: translateX(-50%) translateY(0);
  }
}


</style>