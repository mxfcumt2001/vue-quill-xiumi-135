<template>
  <div class="quill-box" :style="{ height: fullheight + 'px'}">
    <a-modal v-model="visible" title="秀米" width="90%" :footer="null" :maskClosable="false" :centered="true" :keyboard="false">
      <div v-if="visible">
        <iframe src="./pluging/xiumi-ue-dialog-v5.html" frameborder="0" width="100%" :height="(fullheight - 150)+'px'" id="xiumiIframe"></iframe>
      </div>
    </a-modal>
    <a-modal v-model="visible2" title="135编辑器" width="90%" :footer="null" :maskClosable="false" :centered="true" :keyboard="false">
      <div v-if="visible2">
        <iframe src="./pluging/135EditorDialogPage.html" frameborder="0" width="100%" :height="(fullheight - 150)+'px'" id="xiumiIframe"></iframe>
      </div>
    </a-modal>
    <div class="quill-editor">
      <div id="toolbar" slot="toolbar">
        <!-- Add a bold button -->
        <button class="ql-bold" title="加粗"></button>
        <!-- Add a 秀米 button -->
        <button id="custom-button-xiumi" title="秀米" @click="showModal"></button>
        <!-- Add a 135 button -->
        <button id="custom-button-135" title="135编辑器" @click="showModal2"></button>
      </div>
      <div class="editor" ref="editor"></div>
    </div>
  </div>
</template>

<script>
  // 引入原始组件
  import * as Quill from 'quill'
  // 引入核心样式和主题样式
  import 'quill/dist/quill.core.css'
  import 'quill/dist/quill.snow.css'
  // 引入自定义blot
  import blotSelect from './components/blot'
  blotSelect(Quill)

export default {
  name: 'quill-xiumi-135',
  props: {
    // 用于双向绑定
    content: String,
    value: {
      type: String,
      default: ''
    }
  },
  data() {
    return {
      visible: false,
      visible2: false,
      selection: {}, // 光标位置
      fullheight: document.documentElement.clientHeight, // 给quill容器设置了个高度
      quill: null, // 待初始化的编辑器
      options: {
          theme: 'snow', // quill主题
          modules: {
            toolbar: "#toolbar",  // 工具栏的具体配置
          },
          placeholder: '请输入内容...' // quill 2.0.0-dev版本占位符有bug，注意避坑
        }
    }
  },
  mounted () {
    // 初始化编辑器
    this._initEditor()
    // 暴露方法绑定到window上，给public\pluging\xiumi-ue-dialog-v5.html使用
    window.setRichText = this.setRichText
  },
  methods:{
    // 初始化编辑器
    _initEditor () {
      // 获取编辑器的DOM容器
      let editorDom = this.$el.querySelector('.editor')
      // 初始化编辑器
      this.quill = new Quill(editorDom, this.options)
      // 监听光标位置
      this.quill.on('selection-change', range => {
        this.selection = this.quill.getSelection()
        if (!range) {
          this.$emit('blur', this.quill)
        } else {
          this.$emit('focus', this.quill)
        }
      })
      // 双向绑定代码 v-model
      this.quill.on('text-change', () => {
          this.emitChange()
          this.selection = this.quill.getSelection()
        })
      // 插入内容
      this.firstSetHtml()
      // 粘贴板监听
      this.listenPaste()
      this.$emit('ready', this.quill)
    },
    // 回显内容时检查秀米代码
    firstSetHtml() {
      // value 为回显内容
      if(this.value) {
        // 判断是否有秀米和或135元素
        if(this.value.indexOf('xiumi.us') > -1 || this.value.indexOf('135editor.com') > -1 ) {
          let originNode =  new DOMParser().parseFromString(this.value,'text/html').body.childNodes
          this.nodesInQuill(originNode)
        } else {
          // 正常插入
          this.quill.clipboard.dangerouslyPasteHTML(this.value)
        }
      }
    },
    // 根据node类型分发处理
    nodesInQuill(originNode) {
      for(let i = originNode.length - 1; i >= 0; i --) {
        if(originNode[i].localName === 'section') {
          // 秀米类型代码，走新blot
          this.setRichText(originNode[i].outerHTML, 0)
        } else {
          // 正常插入
          this.quill.clipboard.dangerouslyPasteHTML(0, originNode[i].outerHTML)
        }
      }
    },
    // 监听粘贴板
    listenPaste() {
      document.querySelector('.quill-editor').addEventListener('paste', (e) => {
        const msg = (e.clipboardData || window.clipboardData).getData('text/html') // 获取粘贴板文本
        if(msg) { // 如果为空，则不拦截
          e.preventDefault(); // 阻止复制动作
          e.stopPropagation(); // 阻止冒泡
          const value = new DOMParser().parseFromString(msg,'text/html').body.childNodes // 转成nodes
          this.nodesInQuill(value)
        }
      })
    },
    // 更新text-change
    emitChange() {
      // 获取到quill 根dom中的html
      let html = this.$refs.editor.children[0].innerHTML
      const quill = this.quill
      const text = this.quill.getText()
      if (html === '<p><br></p>') html = ''
      // v-model相关
      this.$emit('input', html)
      this.$emit('change', { html, text, quill })
      // 返回quill中文本长度
      // bug注意：这个方法无法计算秀米代码的中的文字长度！
      this.$emit("getConetntLength", this.quill.getLength())
    },
    setRichText(e, t) {
      // 插入位置
      const index = this.selection?this.selection.index: 0
      // 插入数据方法（位置，blot，数据）
      this.quill.insertEmbed(t || index, 'AppPanelEmbed', e)
      // 插入成功后，不管有没有用到a-modal，设置为false就对了
      this.visible = false
      this.visible2 = false
    },
    // 秀米 modal
    showModal() {
      this.visible = true
    },
    // 135 modal
    showModal2() {
      this.visible2 = true
    }
    
  }
}
</script>

<style>
.quill-editor {
  height: 60%;
}
#toolbar button {
  outline:none
}
#custom-button-xiumi, #custom-button-135 {
  background-size: contain;
  background-repeat: no-repeat;
  height: 16px;
  margin-top: 4px;
  width: 33px;
  padding-right: 8px;
  background-position: center;
  position: relative;
}
#custom-button-xiumi {
  background-image: url('http://xiumi.us/connect/ue/xiumi-connect-icon.png');
}
#custom-button-xiumi:hover {
  background-image: url('http://xiumi.us/connect/ue/xiumi-connect-icon.png');
}
#custom-button-135 {
  background-image: url('http://static.135editor.com/img/icons/editor-135-icon.png');
}
#custom-button-135:hover {
  background-image: url('http://static.135editor.com/img/icons/editor-135-icon.png');
}
</style>
