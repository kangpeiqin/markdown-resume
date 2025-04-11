<template>
  <div id="app">
    <header class="minimal-header">
      <div class="header-left">
        <a href="/">Markdown Resume</a>
      </div>
      <div class="header-right">
        <div v-if="isEditing" class="header-action">
          <a class="header-link" @click="toggleEditMode(false)">预览</a>
        </div>
        <div v-else class="header-action">
          <a class="header-link" @click="toggleEditMode(true)">编辑</a>
        </div>
        <div class="dropdown">
          <a class="header-link">主题</a>
          <div class="dropdown-content">
            <a @click="changeStyle('github')">GitHub</a>
            <a @click="changeStyle('tui')">TUI</a>
          </div>
        </div>
        <div class="dropdown">
          <a class="header-link">导出</a>
          <div class="dropdown-content">
            <a @click="downloadPdf()">PDF</a>
            <a @click="downloadHtml()">HTML</a>
            <a @click="downloadMd()">Markdown</a>
          </div>
        </div>
      </div>
    </header>
    
    <!-- 编辑器视图 -->
    <div v-if="isEditing" class="editor minimal-editor">
      <editor
        ref="editor"
        height="calc(100vh - 84px)" 
        :previewStyle="previewStyle"
        :useCommandShortcut="useCommandShortcut"
        :initialValue="mdContent"
        @change="onMdContentChange()"
      />
    </div>
    
    <!-- 简历预览视图 (A4尺寸) -->
    <div v-else class="resume-preview minimal-preview">
      <div class="a4-page" v-html="htmlContent"></div>
    </div>
    
    <!-- 添加页脚 -->
    <footer class="app-footer minimal-footer">
      <div class="footer-content">
        <p>© {{ new Date().getFullYear() }} Markdown Resume</p>
      </div>
    </footer>
  </div>
</template>

<!-- 脚本部分保持不变 -->
<script>
import "@toast-ui/editor/dist/toastui-editor.css";
import { Editor } from "@toast-ui/vue-editor";
import initContent from "./assets/initContent";
import fileDownload from "js-file-download";
import printJS from "print-js";
import "./assets/app.css";
import "./assets/preview.css"; // 添加这一行导入预览样式
import style from "./assets/customized/customizedTheme";

export default {
  name: "App",
  data() {
    return {
      theme: "github",
      isEditing: false,
      mdContent: "",
      htmlContent: "",
      useCommandShortcut: true,
      previewStyle: window.innerWidth > 700 ? "vertical" : "tab",
      localMd: window.localStorage.getItem("mdContent"),
    };
  },
  components: {
    editor: Editor,
  },
  methods: {
    toggleEditMode(isEditing) {
      this.isEditing = isEditing;
      if (!isEditing) {
        // 从编辑模式切换到预览模式时，更新HTML内容
        this.updateHtmlContent();
      }
    },
    updateHtmlContent() {
      if (this.$refs.editor) {
        this.htmlContent = this.$refs.editor.invoke("getHTML");
      }
    },
    downloadPdf: function () {
      let content = this.isEditing ? this.$refs.editor.invoke("getHTML") : this.htmlContent;
      let decorate = style.getOutline({
        content: content,
        css: style.themes[this.theme].pdfCss,
      });
      printJS({
        type: "raw-html",
        css: "",
        scanStyles: true,
        printable: decorate,
        targetStyles: ["*"],
        documentTitle: "&nbsp",
      });
    },
    downloadHtml: function () {
      let content = this.isEditing ? this.$refs.editor.invoke("getHTML") : this.htmlContent;
      fileDownload(content, "resume.html");
    },
    downloadMd: function () {
      let content = this.isEditing ? this.$refs.editor.invoke("getMarkdown") : this.mdContent;
      fileDownload(content, "resume.md");
    },
    changeStyle: function (name) {
      style.removeStyleSheet(this.theme);
      this.theme = name;
      style.addEditorStyle(this.theme);
      
      // 如果在预览模式，需要更新HTML内容以应用新样式
      if (!this.isEditing) {
        this.updateHtmlContent();
      }
    },
    onMdContentChange: function () {
      if (this.$refs.editor) {
        this.mdContent = this.$refs.editor.invoke("getMarkdown");
        this.htmlContent = this.$refs.editor.invoke("getHTML");
        window.localStorage.setItem("mdContent", this.mdContent);
      }
    },
    initializeContent() {
      try {
        // 创建一个临时DOM元素
        const tempDiv = document.createElement('div');
        document.body.appendChild(tempDiv);
        
        // 创建临时编辑器实例
        const tempEditor = new Editor({
          el: tempDiv,
          initialValue: this.mdContent,
          previewStyle: 'vertical',
          height: '0px'
        });
        
        // 确保编辑器已完全初始化后再获取HTML
        setTimeout(() => {
          this.htmlContent = tempEditor.getHTML();
          // 清理临时DOM元素
          tempDiv.remove();
        }, 200);
      } catch (error) {
        console.error('初始化HTML内容失败:', error);
        // 使用简单的转换作为备用方案
        this.htmlContent = this.simpleMarkdownToHtml(this.mdContent);
      }
    },
    
    // 添加一个简单的Markdown到HTML转换函数作为备用
    simpleMarkdownToHtml(markdown) {
      // 简单处理标题、段落和列表
      let html = markdown
        .replace(/^# (.*$)/gm, '<h1>$1</h1>')
        .replace(/^## (.*$)/gm, '<h2>$1</h2>')
        .replace(/^### (.*$)/gm, '<h3>$1</h3>')
        .replace(/\*\*(.*?)\*\*/g, '<strong>$1</strong>')
        .replace(/\*(.*?)\*/g, '<em>$1</em>')
        .replace(/\n- (.*)/g, '<ul><li>$1</li></ul>')
        .replace(/\[([^\]]+)\]\(([^)]+)\)/g, '<a href="$2">$1</a>');
      
      // 处理段落
      const lines = html.split('\n');
      let result = '';
      for (let line of lines) {
        if (!line.startsWith('<h') && !line.startsWith('<ul>') && line.trim() !== '') {
          line = '<p>' + line + '</p>';
        }
        result += line;
      }
      
      return result;
    }
  },
  created() {
    // 设置初始内容
    this.mdContent = this.localMd || initContent;
    
    // 添加样式
    style.addEditorStyle(this.theme);
    
    // 预先设置一个简单的HTML内容，以防initializeContent失败
    this.htmlContent = this.simpleMarkdownToHtml(this.mdContent);
  },
  mounted() {
    // 在DOM挂载后初始化HTML内容
    this.$nextTick(() => {
      this.initializeContent();
    });
    
    // 监听窗口大小变化，调整预览样式
    window.addEventListener('resize', () => {
      this.previewStyle = window.innerWidth > 700 ? "vertical" : "tab";
    });
  },
  beforeDestroy() {
    // 移除事件监听器
    window.removeEventListener('resize', () => {});
  }
};
</script>

<style>

</style>
