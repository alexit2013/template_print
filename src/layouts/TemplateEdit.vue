<template>
    <div>
        <div class="template-edit-page">
            <div class="breadcrumb">
                <router-link class="breadcrumb-item" :to="{ name: 'Index' }" exact>返回上一级</router-link>
                <span class="breadcrumb-item">工作</span>
                <span class="breadcrumb-item">打印模板</span>
                <span class="breadcrumb-item">编辑{{ templateType }}</span>
            </div>
            <div class="template-edit-content">
                <TemplateEditor ref="templateEditor" :propList="propList" :template="template" :templateData="templateData" @openBackConfirmDialog="openBackConfirmDialog" @openDeleteConfirmDialog="openDeleteConfirmDialog" @openComponentRangeOutsideAlertDialog="openComponentRangeOutsideAlertDialog" @preview="previewTemplate" @loadQualityTemplateData="loadQualityTemplateData" @openLoadLabelTemplateDataDialog="openLoadLabelTemplateDataDialog"></TemplateEditor>    
            </div>
            <ConfirmDialog ref="backConfirmDialog" title="返回" content="还未保存，是否确认要退出？" :onConfirmHandler="backConfirmHandler"></ConfirmDialog>
            <ConfirmDialog ref="deleteConfirmDialog" title="删除" content="确认删除选中的组件？" :onConfirmHandler="deleteConfirmHandler"></ConfirmDialog>
            <AlertDialog ref="componentRangeOutsideAlertDialog" title="警告" content="打印时不打印画布大小以外的数据"></AlertDialog>
            <TemplatePreviewDialog ref="templatePreviewDialog" @print="printTemplate" :canvas="canvas" :templateData="templateData" :pageNumber="pageNumber" @close="templatePreviewDialogCloseHandler"></TemplatePreviewDialog>
            <LoadLabelTemplateDataDialog ref="loadLabelTemplateDataDialog" @loadLabelTemplateData="loadLabelTemplateData"></LoadLabelTemplateDataDialog>
        </div>
        <TemplatePreviewCanvasComponent v-show="isPreview" class="template-print-canvas" :isPrintCanvas="true" :canvas="canvas" :templateData="templateData" v-for="i in pageNumber" :page="i"></TemplatePreviewCanvasComponent>
    </div>
</template>

<script>
import {
    mapState
} from 'vuex'
import Vue from 'vue'
import TemplateEditor from '../components/TemplateEditor'
import ConfirmDialog from '../components/ConfirmDialog'
import AlertDialog from '../components/AlertDialog'
import TemplatePreviewDialog from '../components/TemplatePreviewDialog'
import LoadLabelTemplateDataDialog from '../components/LoadLabelTemplateDataDialog'
import TemplatePreviewCanvasComponent from '../components/TemplatePreviewCanvas'

export default {
    name: 'TemplateEditPage',
    data(){
        return {
            templateId: this.$route.query.templateId,
            canvas: {
                percentage: 100,
                width: 0,
                height: 0,
                backgroundImage: '',
                rotateDeg: 0,
                components: [
                    
                ],
            },
            templateData: {
                productList: []
            },
            isPreview: false,
        }
    },
    computed: {
        //打印总页数
        pageNumber(){
            let number = 0
            this.canvas.components.forEach(component => {
                if(component.type == 'ItemListComponent'){
                    number = component.data.number
                }
            })
            //如果没有list，则返回一页
            if(!number){
                return 1
            } else {
                //否则返回页数，没有数据则打印1页
                return Math.ceil(this.templateData.productList.length/number) || 1
            }
        },
        templateType(){
            return this.template.type == 1 ? '质保单':'标签'
        },
        //返回state的属性列表和模板详情到编辑器
        ...mapState({
            propList: state => state.template.propList,
            template: state => state.template.detail,
        })
    },
    components:{
        TemplateEditor,
        ConfirmDialog,
        AlertDialog,
        TemplatePreviewDialog,
        LoadLabelTemplateDataDialog,
        TemplatePreviewCanvasComponent,
    },
    methods:{
        //键盘处理函数，处理删除，复制，粘贴组件
        keydownHandler(e){
            if(!(document.activeElement instanceof HTMLInputElement) && !(document.activeElement instanceof HTMLTextAreaElement)){
                switch(e.keyCode){
                    case 8:
                    case 46:
                        this.$refs.templateEditor.$emit('delete_keyup', e)
                        break
                    case 67:
                        if(e.ctrlKey || e.metaKey){
                            this.$refs.templateEditor.$emit('copy_keyup', e)
                        }
                        break
                    case 86:
                        if(e.ctrlKey || e.metaKey){
                            this.$refs.templateEditor.$emit('paste_keyup', e)
                        }
                        break
                    case 89:
                        if(e.ctrlKey || e.metaKey){
                            this.$refs.templateEditor.$emit('redo_keyup', e)
                        }
                        break
                    case 90:
                        if(e.ctrlKey || e.metaKey){
                            this.$refs.templateEditor.$emit('undo_keyup', e)
                        }
                        break
                    case 37:
                        this.$refs.templateEditor.$emit('move_keyup', 'left')
                        break
                    case 38:
                        this.$refs.templateEditor.$emit('move_keyup', 'up')
                        break
                    case 39:
                        this.$refs.templateEditor.$emit('move_keyup', 'right')
                        break
                    case 40:
                        this.$refs.templateEditor.$emit('move_keyup', 'down')
                        break
                }
                e.preventDefault()
            }
            return false
        },
        mouseupHandler(e){
            this.$refs.templateEditor.$emit('mouseup', e)
        },
        openBackConfirmDialog(){
            this.$refs.backConfirmDialog.show()
        },
        openDeleteConfirmDialog(){
            this.$refs.deleteConfirmDialog.show()
        },
        openComponentRangeOutsideAlertDialog(){
            this.$refs.componentRangeOutsideAlertDialog.show()
        },
        backConfirmHandler(){
            this.$refs.backConfirmDialog.close()
            this.$router.go(-1)
        },
        deleteConfirmHandler(){
            this.$refs.deleteConfirmDialog.close()
            this.$refs.templateEditor.$emit('delete_confirm')
        },
        //预览模板
        previewTemplate(canvas){
            this.canvas = JSON.parse(JSON.stringify(canvas))
            this.isPreview = true
            this.$refs.templatePreviewDialog.show()
        },
        templatePreviewDialogCloseHandler(){
            this.isPreview = false
            this.canvas = {
                percentage: 100,
                width: 0,
                height: 0,
                backgroundImage: '',
                rotateDeg: 0,
                components: [
                    
                ],
            }
        },
        //打印模板
        printTemplate(){
            Vue.nextTick(() => {
                window.print()
            })
        },
        //加载数据用于预览效果
        loadQualityTemplateData(){
            this.$store.dispatch('getPrintQualityData', {orderId: Date.now(), isTmp: 1}).then(json => {
                if(json.state == 200){
                    this.templateData = json.data
                }
            })
        },
        openLoadLabelTemplateDataDialog(){
            this.$refs.loadLabelTemplateDataDialog.show()
        },
        loadLabelTemplateData(productType){
            this.$store.dispatch('getPrintLabelData', {isTmp: 1, productType: productType}).then(json => {
                if(json.state == 200){
                    this.templateData = json.data
                    this.$refs.loadLabelTemplateDataDialog.close()
                }
            })
        }
    },
    mounted(){
        //请求模板的详情，然后根据模板类型请求属性作为菜单，并把模板的画布内容初始化到编辑器上
        this.$store.dispatch('previewTemplate', this.templateId).then(json => {
            if(json.state == 200){
                this.$refs.templateEditor.$emit('set_canvas', json.data.content)
                this.$store.dispatch('getPrintMenuData', json.data.type)
            }
        })
        //初始化键盘事件
        window.addEventListener('keydown', this.keydownHandler)
        window.addEventListener('mouseup', this.mouseupHandler)
    },
    destroyed(){
        //移除组件时移除键盘事件
        window.removeEventListener('keydown', this.keydownHandler)
        window.removeEventListener('mouseup', this.mouseupHandler)
    }
}
</script>

<style lang="scss">
@import "../assets/scss/mixin.scss";
@import "../assets/scss/fonts.scss";
@import "../assets/scss/colors.scss";
@media screen {
    .template-edit-page {
        display: block;
        padding: 0 20px;
        @import "../assets/scss/breadcrumb.scss";
        .template-edit-content {
            .template-editor {
                position: absolute;
                top: 60px;
                bottom: 0;
                left: 20px;
                right: 20px;
            }
        }
    }
    .template-print-canvas {
        opacity: 0;
    }  
}

@media print {
    body {
        background-color: #fff;
    }
    .template-edit-page {
        display: none;
    }
    .template-print-canvas {
        opacity: 1;
    }
}
</style>
