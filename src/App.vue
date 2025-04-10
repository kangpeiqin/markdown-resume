<template>
  <div id="app">
    <header class="minimal-header">
      <div class="header-left">
        <a href="/">简历生成器</a>
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
        <p>© 2023 简历生成器</p>
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
/* 全局样式 */
body {
  margin: 0;
  padding: 0;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
  color: #333;
  background-color: #fafafa;
}

#app {
  height: 100vh;
  display: flex;
  flex-direction: column;
}

/* 极简风格的头部 */
.minimal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 20px;
  height: 50px;
  background-color: #fff;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
}

.header-right {
  display: flex;
  align-items: center;
}

/* 修改所有导航链接的样式 */
.header-left a, .header-link {
  color: #000 !important; /* 使用!important确保覆盖其他样式 */
  text-decoration: none;
  font-weight: 600; /* 加粗字体 */
  font-size: 14px;
  cursor: pointer;
}

.header-left a:hover, .header-link:hover {
  color: #007aff !important; /* 确保悬停颜色也能覆盖其他样式 */
}

/* 确保右侧导航项的样式 */
.header-right .header-link {
  color: #000 !important;
  margin: 0 10px;
}

.header-action {
  margin-right: 20px;
}

/* 下拉菜单样式 */
.dropdown {
  position: relative;
  margin-left: 20px;
}

.dropdown-content {
  display: none;
  position: absolute;
  right: 0;
  top: 30px;
  background-color: #fff;
  min-width: 120px;
  border-radius: 4px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  z-index: 10;
  overflow: hidden;
}

.dropdown-content a {
  color: #333;  /* 确保下拉菜单中的文字颜色为深灰色 */
  padding: 10px 15px;
  display: block;
  text-decoration: none;
  font-size: 14px;
  transition: background-color 0.2s;
}

.dropdown-content a:hover {
  background-color: #f5f5f5;
}

.dropdown:hover .dropdown-content {
  display: block;
}

/* A4纸张样式 */
.minimal-preview {
  flex: 1;
  display: flex;
  justify-content: center;
  align-items: flex-start;
  padding: 30px;
  background-color: #f5f5f5;
  overflow-y: auto;
}

.a4-page {
  width: 210mm;
  min-height: 297mm;
  padding: 20mm;
  background-color: white;
  box-shadow: 0 1px 5px rgba(0, 0, 0, 0.1);
  overflow-y: visible;
}

/* 编辑器样式 */
.minimal-editor {
  flex: 1;
}

/* 页脚样式 */
.minimal-footer {
  height: 34px;
  background-color: #fff;
  color: #666;
  display: flex;
  justify-content: center;
  align-items: center;
  border-top: 1px solid #eee;
}

.footer-content p {
  margin: 0;
  font-size: 12px;
}

/* 响应式调整 */
@media screen and (max-width: 800px) {
  .a4-page {
    width: 100%;
    padding: 15px;
  }
  
  .minimal-header {
    padding: 0 10px;
  }
  
  .header-action {
    margin-right: 10px;
  }
  
  .dropdown {
    margin-left: 10px;
  }
}

/* 打印时隐藏页脚和头部 */
@media print {
  .minimal-header, .minimal-footer {
    display: none;
  }
  
  .minimal-preview {
    padding: 0;
    background-color: white;
  }
  
  .a4-page {
    box-shadow: none;
    padding: 0;
    width: 100%;
  }
}
</style>
