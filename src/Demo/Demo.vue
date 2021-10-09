<template>
  <div class="main">

    <!-- Top bar -->
    <vue-file-toolbar-menu :content="menu" class="bar" />

    <!-- Document editor -->
    <vue-document-editor class="editor" ref="editor"
                         :content.sync="content"
                         :overlay="overlay"
                         :zoom="zoom"
                         :page_format_mm="page_format_mm"
                         :page_margins="page_margins"
                         :display="display" />

  </div>
</template>

<script>
  import VueFileToolbarMenu from 'vue-file-toolbar-menu'
  import VueDocumentEditor from '../DocumentEditor/DocumentEditor.vue'
  // import InvoiceTemplate from './InvoiceTemplate.vue';

  let mousedown = false
  let startTd = ""
  // let td_width;
  let x = 0
  let targets = []
  let highlightDown = false
  let highlightStartTd = ""
  // let onHighlight = false

  function startResize(startObj) {
    mousedown = true
    startTd = startObj
    // td_width = startTd.width;
    x = event.clientX
    targets = getResizeTargets()
  }
  //리사이즈 target 구하기 (target: [ index, width ])
  function getResizeTargets() {
    if (mousedown) {
      //해당 td의 colspan을 계산하여 실제 순서(k)를 구함
      let k = 0
      let nowTds = startTd.parentElement.children
      for (let i = 0; i < nowTds.length; i++) {
        if (startTd.cellIndex < nowTds[i].cellIndex) {
          break
        } else {
          k = k + nowTds[i].colSpan
        }
      }

      let TRs = startTd.parentElement.parentElement.children
      let targets = []
      for (let i = 0; i < TRs.length; i++) {
        let TR = TRs[i]
        let ck = 0 //현재 td의 k
        let TDs = TR.children
        for (let j = 0; j < TDs.length; j++) {
          let TD = TDs[j]
          ck += TD.colSpan
          if (k <= ck) {
            //target td의 현재 cellIndex, width 를 저장
            targets.push({ index: TD.cellIndex, width: parseInt(TD.width) })
            break
          }
        }
      }
      // console.log("targets", targets)
      return targets
    }
  }
  function resizeWidth() {
    if (mousedown) {
      //distX 구하기
      //width가 음수가 되지 않고 하한(10px)을 유지하도록 distX 값을 조절
      let distX = event.x - x;
      let min = 100000
      let minI = -1
      //음수 width 발생을 막고자
      if (distX < 0) { //속도 향상을 위해 distX가 음수일 경우만
        for (let i = 0; i < targets.length; i++) {
          //targets의 최소 width
          if (targets[i].width + distX < min) {
            min = targets[i].width + distX
            minI = i
          }
          //최소 width 가 10 이하면 distX를 증가시켜 최소 10 유지
          if (min < 10) {
            distX = 10 - targets[minI].width
          }
        }
      }

      //resize
      let nowTrs = startTd.parentElement.parentElement.children
      for (let i = 0; i < targets.length; i++) {
        nowTrs[i].children[targets[i].index].width = targets[i].width + distX
      }
    }
  }
  function stopResize(){
    mousedown = false
    startTd = ''
  }
  function isCellLeft(obj){
    if(event.offsetX < 5 && obj.cellIndex!=0)
      return true;
    else
      return false;
  }
  function isCellRight(obj){
    if(event.offsetX > obj.width-4)
      return true;
    else
      return false;
  }
  function resetHighlight() {
    let tables = document.getElementsByClassName("resize-table")
    for (let table of tables) {
      // let TDs = table.getElementsByClassName("highlighted")
      let TDs = table.getElementsByTagName("td")
      for (let TD of TDs) {
        TD.classList.remove("highlighted")
      }
    }
    highlightDown = false
  }

  //realspace의 높이를 spacer에 반영해줌
  //결재선(realspacer)과 content 가 서로 겹치지 않고, 페이징 반영이 잘되도록 하는 조치
  function refreshSpacer() {
    let spacer = document.getElementById("spacer")
    let realSpacer = document.getElementById("real-space")
    spacer.style.height = realSpacer.clientHeight + "px"
  }

  document.onmousedown = function() {
    try {

      //마우스 클릭 시, 마다 spacer 높이를 재설정 해줌...
      //TODO: 이렇게 자주 할 필요 있을까.. 있을 수도 있음
      refreshSpacer()

      let eventedTd = window.event.srcElement
      if (eventedTd.className.toUpperCase() === "COLRESIZE") {
        //셀 왼쪽 border에 위치, 커서 이동
        if ( isCellLeft(eventedTd) ) {
          eventedTd = eventedTd.parentNode.childNodes[eventedTd.cellIndex-1]
          startResize(eventedTd)
          return
        }
        //셀 오른쪽 border에 위치
        if ( isCellRight(eventedTd) ) {
          startResize(eventedTd)
          return
        }
        //border 부근이 아닌 셀 중앙
        highlightDown = true
        highlightStartTd = eventedTd
        // console.log("highlightDown", highlightDown)
        return
      }
      // 표 외부 클릭시 셀선택 해제
      highlightDown = false
      resetHighlight()

    } catch(e) {
      // console.log(e)
    }
  }

  document.onmousemove = function() {
    try {
      let eventedTd = window.event.srcElement;

      //COLRESIZE 위에 롤오버
      if(eventedTd.className.toUpperCase() === "COLRESIZE" || startTd !== "") {
        //border 위에 롤오버
        if( isCellLeft(eventedTd) || isCellRight(eventedTd) ) {
          eventedTd.style.cursor = "col-resize"
        } else {
          eventedTd.style.cursor = ""
          //셀선택
          if (highlightDown) {
            //선택(highlight 처리)이 시작되려면,
            //선택이 시작되는 td에서 mouse down 후, 다른 셀로 mouse move가 발생해야 함
            //즉, 시작되는 셀 안에서 mouse down + move가 있는 거만으로 선택되지 않음
            //이렇게 하는 이유는 시작되는 셀 안에서 text 선택이 가능하게 하기 위함임
            if (highlightStartTd !== eventedTd) { //이동이 있으면..
              eventedTd.classList.add("highlighted")
              highlightStartTd.classList.add("highlighted")
              //이 과정에서 선택된 텍스트 블록은 해제
              window.getSelection().removeAllRanges()
            }
          }
        }

        if (mousedown) {
          resizeWidth(targets)
        }

      } else {
        eventedTd.style.cursor = ""
      }
    } catch(e) {
      // console.log(e)
    }
  }

  document.onmouseup = function() {
    try {
      let now_mouseup = window.event.srcElement;

      //셀선택 해제
      highlightDown = false

      //리사이즈 종료
      stopResize(now_mouseup)
    } catch(e) {
      // console.log(e)
    }
  }

  document.onselectstart = function() {
    try {
      if (startTd !== "") {
        return false
      }
      // if (highlightDown) {
      //   return false
      // }
    } catch(e) {
      // console.log(e)
    }
  }

  // window.document.oncontextmenu = new Function("return false"); window.document.onselectstart = new Function("return false"); window.document.ondragstart = new Function("return false");
  //
  // 출처: https://solbel.tistory.com/60 [개발자의 끄적끄적]


  // //리사이즈시작
  // document.addEventListener("mousedown", mouse_down)
  //
  // //리사이즈
  // document.addEventListener("mousemove", mouse_move)
  //
  // //리사이즈종료
  // document.addEventListener("mouseup", mouse_up)
  //
  // //리사이즈 도중 텍스트 선택 금지
  // document.addEventListener("selectstart", select_start)


  export default {
    components: { VueDocumentEditor, VueFileToolbarMenu },

    data () {
      return {
        // This is where the pages content is stored and synced
        content: [
          ''
        ],
        zoom: 0.8,
        zoom_min: 0.10,
        zoom_max: 5.0,
        page_format_mm: [210, 297],
        page_margins: "10mm 15mm",
        display: "grid", // ["grid", "vertical", "horizontal"]
        mounted: false, // will be true after this component is mounted
        undo_count: -1, // contains the number of times user can undo (= current position in content_history)
        content_history: [], // contains the content states for undo/redo operations

        col_element: "",
        next_element: "",
        cursorStart: 0,
        dragStart: false,
        width: "",
        th_width: "",
        next_width: undefined,
        next_height: "",
        resize: "",
        resize_left: "",
        table_wt: "",
        resizeCheck: "",
        container: "", //document.getElementById("container"),
        table: "", //document.getElementById("table_resize"),
        table_th: "", //table.getElementsByTagName("th"),
        bodyRect: "", //document.body.getBoundingClientRect()

        // container.style.position = "relative";
      }
    },

    created () {
      // Initialize gesture flags
      let start_zoom_gesture = false;
      let start_dist_touch = false;
      let start_zoom_touch = false;

      // Manage ctrl+scroll zoom (or trackpad pinch)
      window.addEventListener("wheel", (e) => {
        if(e.ctrlKey){
          e.preventDefault();
          this.zoom = Math.min(Math.max(this.zoom - e.deltaY * 0.01, this.zoom_min), this.zoom_max);
        }
      }, { passive: false });

      // Manage trackpad pinch on Safari
      window.addEventListener("gesturestart", (e) => {
        e.preventDefault();
        start_zoom_gesture = this.zoom;
      });
      window.addEventListener("gesturechange", (e) => {
        e.preventDefault();
        if(!start_zoom_touch){
          this.zoom = Math.min(Math.max(start_zoom_gesture * e.scale, this.zoom_min), this.zoom_max);
        }
      });
      window.addEventListener("gestureend", () => {
        start_zoom_gesture = false;
      });

      // Manage pinch to zoom for touch devices
      window.addEventListener("touchstart", (e) => {
        if (e.touches.length == 2) {
          e.preventDefault();
          start_dist_touch = Math.hypot(
                  e.touches[0].pageX - e.touches[1].pageX,
                  e.touches[0].pageY - e.touches[1].pageY
          );
          start_zoom_touch = this.zoom;
        }
      }, { passive: false });
      window.addEventListener("touchmove", (e) => {
        if (start_dist_touch && start_zoom_touch) {
          e.preventDefault();
          let zoom = start_zoom_touch * Math.hypot(
                  e.touches[0].pageX - e.touches[1].pageX,
                  e.touches[0].pageY - e.touches[1].pageY
          ) / start_dist_touch;
          this.zoom = Math.min(Math.max(zoom, this.zoom_min), this.zoom_max);
        }
      }, { passive: false });
      window.addEventListener("touchend", () => {
        start_dist_touch = false;
        start_zoom_touch = false;
      }, { passive: false });

      // Manage history undo/redo events
      const manage_undo_redo = (e) => {
        switch(e && e.inputType){
          case "historyUndo": e.preventDefault(); e.stopPropagation(); this.undo(); break;
          case "historyRedo": e.preventDefault(); e.stopPropagation(); this.redo(); break;
        }
      }
      window.addEventListener("beforeinput", manage_undo_redo);
      window.addEventListener("input", manage_undo_redo); // in case of beforeinput event is not implemented (Firefox)

      // If your component is susceptible to be destroyed, don't forget to
      // use window.removeEventListener in the Vue.js beforeDestroy handler
    },

    mounted () { this.mounted = true; },

    computed: {

      // This is the menu content
      menu () {
        return [
          // Main commands
          { text: "New", title: "New", icon: "description", click: () => { if(confirm("This will create an empty document. Are you sure?")){ this.content = [""]; this.resetContentHistory(); } } },
          { text: "Print", title: "Print", icon: "print", click: () => window.print() },

          { is: "spacer" },

          // Undo / redo commands
          { title: "Undo", icon: "undo", disabled: !this.can_undo, hotkey: this.isMacLike ? "command+z" : "ctrl+z", click: () => this.undo() },
          { title: "Redo", icon: "redo", disabled: !this.can_redo, hotkey: this.isMacLike ? "shift+command+z" : "ctrl+y", click: () => this.redo() },

          { is: "spacer" },

          // Rich text menus
          { icon: "format_align_left", title: "Align left", active: this.isLeftAligned, disabled: !this.current_text_style, hotkey: this.isMacLike ? "shift+command+l" : "ctrl+shift+l", click: () => document.execCommand("justifyLeft") },
          { icon: "format_align_center", title: "Align center", active: this.isCentered, disabled: !this.current_text_style, hotkey: this.isMacLike ? "shift+command+e" : "ctrl+shift+e", click: () => document.execCommand("justifyCenter") },
          { icon: "format_align_right", title: "Align right", active: this.isRightAligned, disabled: !this.current_text_style, hotkey: this.isMacLike ? "shift+command+r" : "ctrl+shift+r", click: () => document.execCommand("justifyRight") },
          { icon: "format_align_justify", title: "Justify content", active: this.isJustified, disabled: !this.current_text_style, hotkey: this.isMacLike ? "shift+command+j" : "ctrl+shift+j", click: () => document.execCommand("justifyFull") },
          { is: "separator" },
          { icon: "format_bold", title: "Bold", active: this.isBold, disabled: !this.current_text_style, hotkey: this.isMacLike ? "command+b" : "ctrl+b", click: () => document.execCommand("bold") },
          { icon: "format_italic", title: "Italic", active: this.isItalic, disabled: !this.current_text_style, hotkey: this.isMacLike ? "command+i" : "ctrl+i", click: () => document.execCommand("italic") },
          { icon: "format_underline", title: "Underline", active: this.isUnderline, disabled: !this.current_text_style, hotkey: this.isMacLike ? "command+u" : "ctrl+u", click: () => document.execCommand("underline") },
          { icon: "format_strikethrough", title: "Strike through", active: this.isStrikeThrough, disabled: !this.current_text_style, click: () => document.execCommand("strikethrough") },
          { is: "button-color", type: "compact", menu_class: "align-center", disabled: !this.current_text_style, color: this.curColor, update_color: (new_color) => document.execCommand('foreColor', false, new_color.hex8) },
          { is: "separator" },
          { icon: "format_list_numbered", title: "Numbered list", active: this.isNumberedList, disabled: !this.current_text_style, click: () => document.execCommand("insertOrderedList") },
          { icon: "format_list_bulleted", title: "Bulleted list", active: this.isBulletedList, disabled: !this.current_text_style, click: () => document.execCommand("insertUnorderedList") },
          { html: "<b>H1</b>", title: "Header 1", active: this.isH1, disabled: !this.current_text_style, click: () => document.execCommand('formatBlock', false, '<h1>') },
          { html: "<b>H2</b>", title: "Header 2", active: this.isH2, disabled: !this.current_text_style, click: () => document.execCommand('formatBlock', false, '<h2>') },
          { html: "<b>H3</b>", title: "Header 3", active: this.isH3, disabled: !this.current_text_style, click: () => document.execCommand('formatBlock', false, '<h3>') },
          { icon: "format_clear", title: "Clear format", disabled: !this.current_text_style, click () { document.execCommand('removeFormat'); document.execCommand('formatBlock', false, '<div>'); } },
          { icon: "splitscreen", title: "Page break", disabled: !this.current_text_style, click: () => this.insertPageBreak() },
          { html: "TABLE", title: "TABLE", disabled: !this.current_text_style, click: () => this.makeTable() },

          { is: "spacer" },

          { // Format menu
            text: this.current_format_name,
            title: "Format",
            icon: "crop_free",
            chevron: true,
            menu: this.formats.map(([text, w, h]) => {
              return {
                text,
                active: (this.page_format_mm[0] == w && this.page_format_mm[1] == h),
                click: () => { this.page_format_mm = [w, h]; }
              }
            }),
            menu_width: 80,
            menu_height: 280
          },
          { // Margins menu
            text: this.current_margins_name,
            title: "Margins",
            icon: "select_all",
            chevron: true,
            menu: this.margins.map(([text, value]) => {
              return {
                text: text+" ("+value+")",
                active: (this.page_margins == value),
                click: () => { this.page_margins = value; }
              }
            }),
            menu_width: 200,
            menu_class: "align-center"
          },
          { // Zoom menu
            text: Math.floor(this.zoom * 100) + "%",
            title: "Zoom",
            icon: "zoom_in",
            chevron: true,
            menu: [
              ["200%", 2.0],
              ["150%", 1.5],
              ["125%", 1.25],
              ["100%", 1.0],
              ["75%", 0.75],
              ["50%", 0.5],
              ["25%", 0.25]
            ].map(([text, zoom]) => {
              return {
                text,
                active: this.zoom == zoom,
                click: () => { this.zoom = zoom; }
              }
            }),
            menu_width: 80,
            menu_height: 280,
            menu_class: "align-center"
          },
          { // Display mode menu
            title: "Display",
            icon: this.display == "horizontal" ? "view_column" : (this.display == "vertical" ? "view_stream" : "view_module"),
            chevron: true,
            menu: [{
              icon: "view_module",
              active: this.display == "grid",
              click: () => { this.display = "grid"; }
            }, {
              icon: "view_column",
              active: this.display == "horizontal",
              click: () => { this.display = "horizontal"; }
            }, {
              icon: "view_stream",
              active: this.display == "vertical",
              click: () => { this.display = "vertical"; }
            }],
            menu_width: 55,
            menu_class: "align-right"
          }
        ]
      },

      // Formats management
      current_format_name () {
        const format = this.formats.find(([, width_mm, height_mm]) => (this.page_format_mm[0] == width_mm && this.page_format_mm[1] == height_mm));
        return format ? format[0] : (this.page_format_mm[0]+"mm x "+this.page_format_mm[1]+"mm");
      },
      formats: () => [
        ["A0", 841, 1189],
        ["A0L", 1189, 841],
        ["A1", 594, 841],
        ["A1L", 841, 594],
        ["A2", 420, 594],
        ["A2L", 594, 420],
        ["A3", 297, 420],
        ["A3L", 420, 297],
        ["A4", 210, 297],
        ["A4L", 297, 210],
        ["A5", 148, 210],
        ["A5L", 210, 148],
        ["A6", 105, 148],
        ["A6L", 148, 105]
      ],

      // Margins management
      current_margins_name () {
        const margins = this.margins.find(([, margins]) => (this.page_margins == margins));
        return margins ? margins[0] : margins[1];
      },
      margins: () => [
        ["Medium", "20mm"],
        ["Small", "15mm"],
        ["Slim", "10mm 15mm"],
        ["Tiny", "5mm"]
      ],

      // Current text style management
      current_text_style () { return this.mounted ? this.$refs.editor.current_text_style : false; },
      isLeftAligned () { return ["start", "left", "-moz-left"].includes(this.current_text_style.textAlign); },
      isRightAligned () { return ["end", "right", "-moz-right"].includes(this.current_text_style.textAlign); },
      isCentered () { return ["center", "-moz-center"].includes(this.current_text_style.textAlign); },
      isJustified () { return ["justify", "justify-all"].includes(this.current_text_style.textAlign); },
      isBold () {
        const fontWeight = this.current_text_style.fontWeight;
        return fontWeight && (parseInt(fontWeight) > 400 || fontWeight.indexOf("bold") == 0);
      },
      isItalic () { return this.current_text_style.fontStyle == "italic"; },
      isUnderline () { // text-decoration is not overridden by children, so we query the parent stack
        const stack = this.current_text_style.textDecorationStack;
        return stack && stack.some(d => (d.indexOf("underline") == 0));
      },
      isStrikeThrough () { // text-decoration is not overridden by children, so we query the parent stack
        const stack = this.current_text_style.textDecorationStack;
        return stack && stack.some(d => (d.indexOf("line-through") == 0));
      },
      isNumberedList () { return this.current_text_style.isList && this.current_text_style.listStyleType == "decimal"; },
      isBulletedList () { return this.current_text_style.isList && ["disc", "circle"].includes(this.current_text_style.listStyleType); },
      isH1 () { return this.current_text_style.headerLevel == 1; },
      isH2 () { return this.current_text_style.headerLevel == 2; },
      isH3 () { return this.current_text_style.headerLevel == 3; },
      curColor () { return this.current_text_style.color || "transparent"; },

      // Platform management
      isMacLike: () => /(Mac|iPhone|iPod|iPad)/i.test(navigator.platform),

      // Undo / redo flags
      can_undo () { return this.undo_count > 0; },
      can_redo () { return this.content_history.length - this.undo_count - 1 > 0; }
    },

    methods: {

      makeTable() {
        let t = `<table class="resize-table"><tbody><tr><td width=200" colspan="2" class="colresize">가</td><td width="100" class="colresize">다다다</td></tr><tr><td width="100" class="colresize">라라라라</td><td width="100" class="colresize">마마마마 마</td><td width="100" class="colresize">바바바바 바바</td></tr></tbody></table>`;
        document.execCommand("insertHtml", false, t)
        // let resizeTable = document.getElementById("resize-table")
        // console.log("resizeTable", resizeTable)
        // console.log("resizeTable", resizeTable.childElementCount)
        // console.log("resizeTable", resizeTable.children[0].children)
        // this.setTdWidth()
        // this.createResizeDiv()
        // this.initEvents()
      },

      setTdWidth() {

        //초기화
        this.container = document.getElementById("container")
        this.table = document.getElementById("table_resize")
        this.table_th = this.table.getElementsByTagName("th")
        this.bodyRect = document.body.getBoundingClientRect()
        this.container.style.position = "relative";

        let elm_bound = this.table.getBoundingClientRect();
        let table_wt = elm_bound.width;
        let th_length = this.table_th.length;
        this.th_width = table_wt/th_length;

        console.log("table_wt", table_wt, "th_length", th_length, "th_width", this.th_width)
      },

      createResizeDiv() {
        let cont = document.getElementById("container");
        let th_length = this.table_th.length;
        for(let i=1; i<=th_length; i++){
          let yDiv = document.createElement("div");
          yDiv.className = "y_resize tb_resize";
          yDiv.setAttribute("data-resizecol",i);
          let leftPos = (i*this.th_width)+0.5;
          yDiv.style.cssText = "left: "+leftPos+"px;";
          cont.append(yDiv);
        }
      },

      // Page overlays (headers, footers, page numbers)
      overlay (page, total) {
        // Add page numbers on each page
        let html = '<div style="position: absolute; bottom: 8mm; ' + ((page % 2) ? 'right' : 'left') + ': 10mm">Page ' + page + ' of ' + total + '</div>';

        // Add custom headers and footers from page 3
        // if(page >= 3) {
        //   html += '<div style="position: absolute; left: 0; top: 0; right: 0; padding: 3mm 5mm; background: rgba(200, 220, 240, 0.5)"><strong>MYCOMPANY</strong> example.com /// This is a custom header overlay</div>';
        //   html += '<div style="position: absolute; left: 10mm; right: 10mm; bottom: 5mm; text-align:center; font-size:10pt">Copyright (c) 2020 Romain Lamothe, MIT License /// This is a custom footer overlay</div>';
        // }

        // if (page === total) {
        //   html += '<div id="real-space" style="height: 30mm; width: 170mm; background: #FAFAFA; position: absolute; bottom: 20mm" contenteditable="false">&nbsp; 흐히히</div>'
        // }
        return html;
      },

      // Undo / redo functions examples
      undo () { if(this.can_undo){ this._mute_next_content_watcher = true; this.content = this.content_history[--this.undo_count]; } },
      redo () { if(this.can_redo){ this._mute_next_content_watcher = true; this.content = this.content_history[++this.undo_count]; } },
      resetContentHistory () { this.content_history = []; this.undo_count = -1; },

      // Insert page break function example
      async insertPageBreak () {
        // insert paragraph at caret position
        document.execCommand("insertParagraph");

        // insert a marker at caret position (start of the new paragraph)
        const marker = "###PB###"; // must be regex compatible
        document.execCommand("insertText", false, marker);

        // wait for DOM update
        await this.$nextTick();

        // find the marker inside content items and split this content item in two items between the two paragraphs
        // only match root tags (p, div, h1, h2...) to avoid non-root tags like <li>
        const regexp = new RegExp("<(p|div|h\\d)( [^/>]+)*>(<[^/>]+>)*"+marker);
        for(let i = 0; i < this.content.length; i++) {
          const item = this.content[i];
          if(typeof item != "string") continue;
          const match = regexp.exec(item);
          if(match) {
            const tags_open = match[0].slice(0, -marker.length);
            let content_plus_tags_close = item.substr(match.index + match[0].length);
            // insert <br> to empty pages that would not be handled correctly by contenteditable
            if(content_plus_tags_close.indexOf("</") == 0) content_plus_tags_close = "<br>" + content_plus_tags_close;
            this.content.splice(i, 1, item.substr(0, match.index), tags_open + content_plus_tags_close);
            return;
          }
        }

        // if the code didn't return before, the split didn't work (e.g. inside a <li>). just remove the marker from the content
        for(let i = 0; i < this.content.length; i++) {
          const item = this.content[i];
          if(typeof item != "string" || item.indexOf(marker) < 0) continue;
          this.content.splice(i, 1, item.replace(marker, ''));
          break;
        }
      }
    },

    watch: {
      content: {
        immediate: true,
        // Fill undo / redo history stack on user input
        handler (new_content) {
          if(!this._mute_next_content_watcher) { // only update the stack when content is changed by user input, not undo/redo commands
            this.content_history[++this.undo_count] = new_content;
            this.content_history.length = this.undo_count + 1; // remove all redo items
          }
          this._mute_next_content_watcher = false;
        }
      }
    }
  }
</script>

<style>
  html {
    height: 100%;
  }
  body {
    margin: 0;
    font-family: Avenir, Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    color: black;
    background: rgb(248, 249, 250);
  }
  ::-webkit-scrollbar {
    width: 16px;
    height: 16px;
  }
  ::-webkit-scrollbar-track, ::-webkit-scrollbar-corner {
    display: none;
  }
  ::-webkit-scrollbar-thumb {
    background-color: rgba(0, 0, 0, 0.5);
    border: 5px solid transparent;
    border-radius: 16px;
    background-clip: content-box;
  }
  ::-webkit-scrollbar-thumb:hover {
    background-color: rgba(0, 0, 0, 0.8);
  }
</style>
<style scoped>
  .main {
    width: fit-content;
    min-width: 100%;
  }
  .bar {
    position: sticky;
    left: 0;
    top: 0;
    width: calc(100vw - 16px);
    z-index: 1000;
    background: rgba(248, 249, 250, 0.8);
    border-bottom: solid 1px rgb(248, 249, 250);
    backdrop-filter: blur(10px);
    --bar-button-active-color: #188038;
    --bar-button-open-color: #188038;
    --bar-button-active-bkg: #e6f4ea;
    --bar-button-open-bkg: #e6f4ea;
  }
</style>
<style>
  .container{
    position: relative;
    display: inline-block;
  }
  table{
    border-collapse: collapse;
    /*outline: none;*/
    /*table-layout: fixed;*/
    /*width: 500px;*/
  }
  table tr td, table tr th{
    word-wrap: break-word;
    border: 1px solid #000;
    /*min-width: 130px;*/
    padding: 5px;
    position: relative;
    text-align: left;
    box-sizing: border-box;
    /*min-width: 30px;*/
  }
  .y_resize{
    position: absolute;
    top: 0;
    left: 0;
    width: 7px;
    height: 102%;
    background: transparent;
    cursor: col-resize;
    border: 1px solid red;
  }
  .colresize {
  }
  .highlighted {
    background: antiquewhite;
  }
</style>