<template>
    <div class="epub-view-box">

        <div class="epub-view-container">
            <!-- left tools -->
            <div class="epub-view-left" v-if="load">
                <div>
                    <el-button @click="settingModal = true">显示设置</el-button>
                </div>
                <div>
                    <el-button @click="directoryModal = true">目录</el-button>
                </div>
                <div>
                    <el-button @click="searchModal = true">内容搜索</el-button>
                </div>
            </div>
            <!-- content -->
            <div class="epub-view-content" :class="[settingData.value.themeColor]" id="book" :style="{
                '--annotation-color': settingData.value.lineColor
            }"></div>
            <!-- right tools -->
            <div class="epub-view-right" v-if="load">
                <div>
                    <el-button @click="handleNext(false)">上一页</el-button>
                </div>
                <div>
                    <el-button @click="handleNext(true)">下一页</el-button>
                </div>
            </div>
        </div>
        <!-- 显示设置 -->
        <el-dialog v-model="settingModal" title="显示设置" width="512px">
            <el-form label-position="top" label-width="100px" :model="settingData.data">
                <el-form-item label="阅读主题">
                    <el-radio-group v-model="settingData.data.themeColor" label="label position">
                        <el-radio-button :label="item.value" v-for="item in settingData.themeColor"
                            :type="settingData.data.themeColor == item.value ? 'primary' : ''">
                            <span
                                :style="{ width: '32px', height: '32px', 'backgroundColor': item.label, display: 'inline-block' }"></span>
                        </el-radio-button>
                    </el-radio-group>
                </el-form-item>
                <el-form-item label="划线笔记背景色">
                    <el-radio-group v-model="settingData.data.lineColor" label="label position">
                        <el-radio-button :label="item" v-for="item in settingData.lineColor"
                            :type="settingData.data.lineColor == item ? 'primary' : ''">
                            <span
                                :style="{ width: '32px', height: '32px', 'backgroundColor': item, display: 'inline-block' }"></span>
                        </el-radio-button>
                    </el-radio-group>
                </el-form-item>
                <el-form-item label="字号">
                    <el-select v-model="settingData.data.fontSize">
                        <el-option v-for="item in [12, 14, 16, 18, 20, 22, 24]" :key="item" :label="item"
                            :value="item" />
                    </el-select>
                </el-form-item>
                <el-form-item label="阅读模式">
                    <el-select v-model="settingData.data.readModel">
                        <el-option v-for="item in settingData.readModel" :key="item.value" :label="item.label"
                            :value="item.value" />
                    </el-select>
                </el-form-item>
                <el-form-item>
                    <el-button type="primary" @click="handleDisplaySubmit">确定</el-button>
                </el-form-item>
            </el-form>
        </el-dialog>
        <!-- 目录 -->
        <el-dialog v-model="directoryModal" title="目录" width="512px">
            <div style="height:240px;overflow:auto">
                <p v-for="item in menuList" :key="item" style="padding:8px;cursor: pointer;"
                    @click="handleJump(item.href)">
                    {{
                            item.label
                    }}</p>
            </div>

        </el-dialog>
        <!-- 全文搜索 -->
        <el-dialog v-model="searchModal" title="内容搜索" width="512px">
            <el-space>
                <el-input v-model="searchModalData.content" placeholder="请输入搜索内容" />
                <el-button type="primary" @click="handleSearch">搜索</el-button>
            </el-space>
            <div style="height:240px;overflow:auto">
                <p v-for="item in searchModalData.list" :key="item.cfi" style="padding: 8px;"
                    v-html="handleHig(item.excerpt, searchModalData.content)" @click="handleJump(item.cfi)">
                </p>
            </div>
        </el-dialog>

        <!-- 笔记弹窗 -->
        <el-dialog v-model="noteModal" title="笔记" width="512px">
            <p>{{ noteModalData.content }}</p>
            <br />
            <el-input type="textarea" v-model="noteModalData.input"></el-input>
            <br />
            <br />
            <el-button @click="handleAddNote" type="primary">设置笔记</el-button>
            <el-button @click="handleDelNote" type="primary">删除笔记</el-button>
        </el-dialog>

        <!-- dev ------------------------------------------------------------------->
        <span class="dev-btn">
            <el-button type="primary" @click="drawer = true">开发者</el-button>
        </span>
        <el-drawer v-model="drawer" direction="rtl">
            <el-space>
                <el-upload action="" :show-file-list="false" :before-upload="handleDevOpenBook" v-if="!load">
                    <el-button type="primary">选择电子书</el-button>
                </el-upload>
                <el-button type="danger" @click="handleDevDestroy" v-else>销毁电子书</el-button>
            </el-space>

            <div v-if="!load">
                <br />
                <h4>测试书本：</h4>
                <p v-for="item in book_list" :key="item" style="padding:8px 0;">{{ item }} <el-button type="text"
                        @click="handleLoadBook(item)">打开</el-button></p>
            </div>
            <div>
                <p>选中内容后，通过浮动工具栏进行划线 高亮设置</p>
            </div>

        </el-drawer>
    </div>
</template>
<script >
import Epub, { Book, EpubCFI } from "epubjs"
import { ElMessage } from 'element-plus'
let book = null;
let rendition = null;
export default {
    data() {
        return {
            load: false,
            menuList: [],
            settingModal: false,
            settingData: {
                // 阅读主题色
                themeColor: [{
                    value: 'dark',
                    label: '#000'
                }, {
                    value: 'light',
                    label: '#fff'
                }, {
                    value: 'tan',
                    label: 'tan'
                }],
                lineColor: ["#293462", "#8134af", "#dd2a7b", "#feda77"],
                fontSize: [12, 36],
                readModel: [{
                    value: 'horizontal',
                    label: '横向'
                }, {
                    value: 'vertical',
                    label: '竖向'
                }],
                data: {
                    themeColor: 'light',
                    lineColor: '#8134af',
                    fontSize: 16,
                    readModel: 'vertical',
                },
                value: {}

            },
            directoryModal: false,
            searchModal: false,
            searchModalData: {
                content: '',
                list: []
            },
            noteModal: false,
            noteModalData: {
                content: '',
                input: '',
                data: {}
            },
            // dev -------------------------
            drawer: false,
            book_list: [
                "/1.epub",
                "/2.epub",
            ]
        }
    },
    mounted() {

    },
    methods: {
        /**
         * 加载epub
         */
        async handleLoadBook(bookData) {
            book = Epub();
            if (typeof bookData == 'string') {
                await book.open(bookData);
            } else {
                await book.open(bookData, "binary");
            }
            this.drawer = false;
            this.load = true;
            rendition = book.renderTo("book", {
                width: "100%",
                height: "100vh",
                flow: "scrolled",
                spread: 'none',
                // allowScriptedContent: true,
            });
            rendition.themes.register("dark", "themes.css");
            rendition.themes.register("light", "themes.css");
            rendition.themes.register("tan", "themes.css");
            this.handleDisplaySubmit();
            // rendition.hooks.content.register(function (contents, view) {
            //     console.log(contents);
            // })
            rendition.on("started", () => {
                console.log("started");
            })
            rendition.on("attached", () => {
                console.log("attached");
            })
            rendition.on("displayed", () => {
                console.log("-----------------------");

            })
            rendition.on("displayError", () => {
                console.log("displayError");
            })
            rendition.on("rendered", () => {
                console.log("rendered");
            })
            rendition.on("removed", () => {
                console.log("removed");
            })
            rendition.on("resized", () => {
                console.log("resized");
            })
            rendition.on("orientationchange", () => {
                console.log("orientationchange");
            })
            rendition.on("locationChanged", () => {
                console.log("locationChanged");
            })
            rendition.on("relocated", () => {
                console.log("relocated");
            })
            rendition.on("selected", () => {
                console.log("selected");
            })
            rendition.on("markClicked", () => {
                console.log("started");
            })
            rendition.on("keyup", () => {
                console.log("started");
            })


            let init = false;
            rendition.on("rendered", () => {
                let time = 0;
                rendition.getContents()[0].document.addEventListener("selectionchange", function () {
                    clearTimeout(time);
                    time = setTimeout(() => {
                        let Selection = rendition.getContents()[0].window.getSelection();
                        if (!Selection) {
                            return;
                        }
                        let isCollapsed = Selection.isCollapsed;
                        if (isCollapsed) {
                            let el = document.getElementById('tool')
                            if (!el) {
                                return;
                            }
                            // SUCCESS 解决了取消选中后 工具栏未消失问题
                            el.style.display = 'none';
                        }

                    }, 17)
                });
                if (!init) {
                    // 采用追加元素的方式达到浮动工具栏的效果
                    let span = document.createElement("span");
                    span.id = "tool"
                    span.style.position = "absolute"
                    span.style.top = "100px";
                    span.style.left = "100px";
                    span.style.zIndex = "1";
                    span.className = 'tool-span'
                    span.style.display = 'none';
                    let arr = ["高亮", "划线"];
                    let f = document.createDocumentFragment();
                    let that = this;
                    arr.forEach((item, index) => {
                        let _span = document.createElement("span");
                        _span.textContent = item;
                        _span.setAttribute('data-index', index.toString());
                        _span.addEventListener('click', function () {
                            let el = document.getElementById('tool');
                            if (!el) {
                                return;
                            }
                            let id = el.getAttribute('data-id');
                            let index = this.getAttribute('data-index');
                            if (index == '0') {
                                rendition.annotations.highlight(id, {}, that.handleClick, 'highlight', {});
                            } else if (index == '1') {
                                rendition.annotations.underline(id, {}, that.handleClick, 'underline', {});
                            }
                            if (el) {
                                el.style.display = 'none';
                            }
                            rendition.getContents()[0].window.getSelection().removeAllRanges()

                        })
                        f.appendChild(_span);
                    })
                    span.appendChild(f);
                    document.querySelector('.epub-container')?.appendChild(span);
                    init = true;
                }
            })
            rendition.on("selected", (cfiRange, contents) => {
                let ff = new EpubCFI(cfiRange);
                let _top = contents.window.window.frameElement.parentElement.offsetTop
                let _base = ff.toRange(contents.document);
                let arr = _base.getClientRects();
                let { left, top } = arr[0];
                let tool = document.getElementById('tool')
                if (tool) {
                    tool.style.display = 'inline-block';
                    tool.style.left = left + 'px';
                    tool.style.top = top + _top + 'px';
                    tool.setAttribute('data-id', cfiRange);
                }
                // contents.window.getSelection().removeAllRanges();
            })
            // 加载完成后 获取目录
            book.loaded.navigation.then(doc => {
                this.menuList = doc.toc;
            })
            // 显示第一页
            rendition.display();


        },
        /**
         * 上下页
         */
        handleNext(bool = false) {
            if (bool) {
                rendition && rendition.next();
            } else {
                rendition && rendition.prev();
            }
        },
        /**
         * 显示设置
         */
        handleDisplaySubmit() {
            let settingData = this.settingData.data;
            this.settingData.value = JSON.parse(JSON.stringify(settingData));
            rendition.themes.select(settingData.themeColor);
            rendition.themes.fontSize(settingData.fontSize + 'px');
            let flow = this.settingData.data.readModel == 'horizontal' ? 'paginated' : 'scrolled'
            rendition.flow(flow);
            this.settingModal = false;
        },
        async handleSearch() {
            let content = this.searchModalData.content;
            // 全文搜索
            let res = await Promise.all(
                book.spine.spineItems.map(section => section.load(book.load
                    .bind(book))
                    .then(section.find.bind(section, content))
                    .finally(section.unload.bind(section)))
            ).then(results => Promise.resolve([].concat.apply([], results))) // apply通过参数传递results中的值，达到降低维度的效果，一定要用空数组
            console.log(res);
            this.searchModalData.list = res;
        },
        handleHig(text, content) {
            return text.replace(content, `<b style="color:#f29c2b">${content}</b>`);
        },
        handleJump(cfi) {
            rendition.display(cfi);
            this.searchModal = false;
            this.directoryModal = false;
        },
        async handleClick(e) {
            this.noteModal = true;
            let target = e.target;

            let efi = target.getAttribute('data-epubcfi');
            let ff = new EpubCFI(efi);
            let data = JSON.parse(JSON.stringify(this.noteModalData.data));
            let range = await book.getRange(ff);
            // 当前位置信息
            // await rendition.currentLocation()


            if (data[efi]) {
                this.noteModalData.content = range.toString();
                this.noteModalData.input = data[efi];
                this.noteModalData.efi = efi;
            } else {
                this.noteModalData.content = range.toString();
                this.noteModalData.input = "";
                this.noteModalData.efi = efi;
            }

        },
        /**
         * 添加笔记
         */
        handleAddNote() {
            let efi = this.noteModalData.efi;
            this.noteModalData.data[efi] = this.noteModalData.input;
            this.noteModal = false;
        },
        /**
         * 删除笔记
         */
        handleDelNote() {
            let efi = this.noteModalData.efi;
            this.noteModalData.data[efi] = "";
            this.noteModal = false;
            rendition.annotations.remove(efi, "highlight");
        },
        // dev ----------------------------------------
        /**
         * 打开电子书
         */
        handleDevOpenBook(file) {
            if (file.type != 'application/epub') {
                ElMessage({
                    message: '文件格式错误',
                    grouping: true,
                    type: 'danger',
                })
                return;
            }
            if (window.FileReader) {
                var reader = new FileReader();
                reader.onload = (r) => {
                    this.handleLoadBook(r.target.result);
                };
                reader.readAsArrayBuffer(file);
            }
            return false;
        },
        handleDevDestroy() {
            book && book.destroy();
            this.load = false;
        },

    }
}
</script>
<style  >
@import "/themes.css";

.epub-view-box {
    width: 100vw;
    height: 100vh;
    background-color: #ccc;
}

.epub-view {
    width: 100% !important;
}

.epub-view-container {
    width: 960px;
    height: 100%;
    margin: 0 auto;
    box-sizing: border-box;
    background-color: #fff;
    position: relative;
}

.epub-view-left {
    width: 120px;
    height: 100%;
    position: absolute;
    left: 0;
    top: 0;
    transform: translateX(-100%);
    display: flex;
    flex-direction: column;
    align-items: center;
}

.epub-view-left>div {
    padding: 8px 0;
    box-sizing: border-box;
}

.epub-view-left>div .el-button {
    width: 96px;
}

.epub-view-right {
    width: 120px;
    height: 100%;
    position: absolute;
    right: 0;
    top: 0;
    transform: translateX(100%);
    display: flex;
    flex-direction: column;
    justify-content: flex-end;
    align-items: center;
    padding: 24px 0;
    box-sizing: border-box;
}

.epub-view-right>div {
    padding: 8px 0;
    box-sizing: border-box;
}

.epub-view-right>div .el-button {
    width: 96px;
}

.epub-view-content {
    width: 100%;
    height: 100%;
}

.tool-span {
    height: 48px;
    border-radius: 8px;
    transform: translateY(calc(-100%));
    line-height: 48px;
    overflow: hidden;
    transition: all 0.3s ease;
}

.tool-span>span {
    padding: 0 16px;
    height: 100%;
    display: inline-block;
    background-color: #000;
    color: #fff;
    cursor: pointer;
}

.tool-span>span:hover {
    background-color: #333;
}

/* dev */
.dev-btn {
    position: fixed;
    right: 24px;
    bottom: 24px;
}
</style>