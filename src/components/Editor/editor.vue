<template>
  <!-- weui编辑器 -->
  <div class="weui_editor">
    <!-- weui-UI -->
    <div class="weui_edui_wrap">
      <!-- weui工具栏 -->
      <div class="weui_editor_hd">
        <div class="weui_editor_toolbar_box_wrap" ref="toolbar">
          <weuiToolbar
            :toolStates="toolStates"
            :toolValues="toolValues"
            :imageFloatMask="imageMaskVisible"
            :selectOptions="selectOptions"
            :listVisible="listVisible"
            :maskVisible="toolbarMaskVisible"
            @on-editor-execCommand="execCommand"
            @on-maskvisible-change="onMaskVisibleChange"
          ></weuiToolbar>
        </div>
      </div>
      <div class="weui_editor_bd">
        <slot class="weui_editor_extra" name="header"></slot>
        <div class="weui_editor_body">
          <div ref="editor"></div>
        </div>
        <slot class="weui_editor_extra" name="footer"></slot>
      </div>
      <div class="weui_editor_ft"></div>
    </div>
    <slot class="weui_editor_loading" name="loading"></slot>
  </div>
</template>
<script type="text/ecmascript-6">
  import weuiToolbar from '../Toolbar';
  import './editor.stylus';
  import { selectOptions, ueConfig } from '../../config/config';
  import '../../config/ueditor.config';
  import '../../../static/ueditor.all.min';
  import '../../../static/lang/zh-cn/zh-cn';

  export default {
    name: 'weui-ueditor',
    components: { weuiToolbar },
    props: {
      content: {
        type: String,
        default: ''
      },
      config: {
        type: Object,
        default() {
          return {};
        }
      },
      scrollElement: {
        type: String,
      },
      fixedToolbarScrollTop: {
        type: Number,
        default: 0
      },
      fixedToolbarOffset: {
        type: Number,
        default: 0
      }
    },
    computed: {
      selectOptions() {
        return Object.assign({}, selectOptions, this.config.selectOptions);
      },
      ueConfig() {
        return Object.assign({}, ueConfig, this.config.ueditor);
      }
    },
    data() {
      return {
        // 编辑器id
        id: 'weui_edui_' + Math.random(10),
        UE: window.UE || null,
        editor: null,
        _content: '',
        // 工具条状态和数据
        toolStates: {
          // 是否可操作 -1:不可用 0:可用 1:已操作
          undo: -1,
          redo: -1,
          blockquote: 0,
          horizontal: 0,
          removeformat: 0,
          formatmatch: 0,
          bold: 0,
          italic: 0,
          underline: 0,
          indent: 0
        },
        // 子组件的值，必须确保在此注册才能监听动态数据双向绑定
        toolValues: {
          fontsize: '16px',
          forecolor: '',
          backcolor: '',
          justify: '',
          imagefloat: '',
          lineheight: '',
          rowspacingtop: '',
          rowspacingbottom: '',
          insertorderedlist: '',
          insertunorderedlist: ''
        },
        listVisible: !0,
        toolbarMaskVisible: !0,
        imageMaskVisible: !1
      };
    },
    methods: {
      init() {
        console.info('<editor-component> inited');
        setTimeout(() => {
          this.setContent();
          this.$emit('on-editor-ready');
        }, 800);
      },
      setContent(value) {
        this.editor.setContent(value || this.content || this._content || '');
      },
      execCommand(name, value, dir) {
        this.editor.execCommand(name, value, dir);
      },
      onScroll: function onScroll() {
        const scrollElem = document.querySelector(this.scrollElement) || document.body;
        const H = scrollElem.scrollTop;
        const toolbarboxWrapCss = getComputedStyle(this.toolbar.parentNode, false);
        const toolbarCss = getComputedStyle(this.toolbar, false);

        const elemCss = getComputedStyle(this.$el, false);
        const elemHeight = parseInt(elemCss.height, 10)
        const toolbarHeight = parseInt(toolbarboxWrapCss.height, 10)
        const fixedToolbarStopScrollTop = this.fixedToolbarScrollTop + (elemHeight - toolbarHeight)

        try {
          if (H > this.fixedToolbarScrollTop && H < fixedToolbarStopScrollTop) {
            if (!this.toolbar.style.cssText) {
              this.toolbar.style.cssText = `top:${this.fixedToolbarOffset}px;position:fixed;width:${toolbarboxWrapCss.width}`;
              this.toolbar.parentNode.style.cssText = `height:${toolbarCss.height}`;
              // 不知道为什么有时页面加载成功后会误判定进这里，所以干脆在一秒后重新检查一次
              setTimeout(onScroll.bind(this), 1000)
            }
          } else if (H < this.fixedToolbarScrollTop) {
            if (this.toolbar.style.cssText) this.toolbar.style.cssText = '';
            if (this.toolbar.parentNode.style.cssText) this.toolbar.parentNode.style.cssText = '';
          } else if (H > fixedToolbarStopScrollTop) {
            // 从下向上滚动时，如果 wrap 和 toolbar 的 cssText 都是被清空的
            // 滚动到 fixedToolbarStopScrollTop 时页面会跳动一下，再次滚动到
            // fixedToolbarStopScrollTop 之下。如果一直比较缓慢地向上滚动，
            // 那么页面会一直跳动，导致永远无法滚动到页面顶部
            if (this.toolbar.style.cssText) this.toolbar.style.cssText = '';
          }
        } catch (e) {
          console.warn(e);
        }
      },
      getContent(type, fn) {
        type = type || 'Content';
        return this.editor[`get${type}`](fn);
      },
      /**
       * 监听ueditor 编辑器内容更改，返回给editor-component
       * @createdAt 2017-05-15T10:19:22+0800
       * @author yiwuyu
       */
      onContentChange() {
        this.editor.addListener('contentChange', function() {
          this._content = this.editor.getContent();
          this.$emit('contentChange');
        }.bind(this));
      },
      /**
       * 监听编辑器光标选取改变，触发对toolbars的遮罩和状态option切换
       * @createdAt 2017-05-15T10:20:24+0800
       * @author yiwuyu
       */
      onSelectionChange() {
        this.editor.addListener('selectionchange', function() {
          const statelist = Object.keys(this.toolStates);
          const valuelist = Object.keys(this.toolValues);

          for (let item of statelist) {
            this.toolStates[item] = this.editor.queryCommandState(item);
          }
          for (let item of valuelist) {
            if (item.indexOf('rowspacing') > -1) {
              // rowspacing的值比较特别
              this.toolValues[item] = this.editor.queryCommandValue('rowspacing', item.replace('rowspacing', ''));
            } else {
              this.toolValues[item] = this.editor.queryCommandValue(item) + '';
            }
          }
          // 图片浮动
          const dom = this.editor.selection.getRange().getClosedNode();
          if (dom && dom.tagName === 'IMG') {
            this.imageMaskVisible = !1;
          } else {
            this.imageMaskVisible = !0;
          }
        }.bind(this));
      },
      onFocus() {
        this.editor.addListener('focus', function() {
          this.toolbarMaskVisible = !1;
          this.listVisible = !1;
        }.bind(this));
        this.editor.addListener('blur', function() {
          this.listVisible = !0;
        }.bind(this));
      },
      onMaskVisibleChange(val) {
        this.toolbarMaskVisible = !0;
      }
    },
    mounted() {
      this.toolbar = this.$refs.toolbar;
      this.$nextTick(() => {
        this.$refs.editor.id = this.id;
        this.editor = this.UE.getEditor(this.id, this.ueConfig);
        this.editor.ready(() => {
          this.init();
          this.onContentChange();
          this.onSelectionChange();
          this.onFocus();

          const scrollElem = document.querySelector(this.scrollElement) || window;
          scrollElem.addEventListener('scroll', this.onScroll, false);
        });
      });
    },
    beforeDestroy() {
      const scrollElem = document.querySelector(this.scrollElement) || window;
      scrollElem.removeEventListener('scroll', this.onScroll, false);
    },
  };
</script>
