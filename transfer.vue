<template>
    <BaseDialog :visible="isDialog" :footer="isAuth" :width="680" :title="defaultKey.title" @beforeClose="beforeClose" @confirm="confirm">
        <div slot="body">
            <div class="flex-justify-between transfer-tree">
                <div class="treeLeft borders pr">
                    <div class="title">{{ $t('noSelected') }}</div>
                    <el-input :style="{ position: 'absolute', zIndex: 10 }" v-model="searchKey" :placeholder="$t('pleaseEnter', { name: $t('keywords') })"></el-input>
                    <div class="treeData" :style="{ width: '235px', paddingTop: '30px' }">
                        <vue-easy-tree
                            ref="leftTree"
                            class="virtutal-easy-tree checkbox-tree special-tree"
                            node-key="nodeId"
                            height="370px"
                            :data="sourceList"
                            highlight-current
                            accordion
                            :render-content="renderContent"
                            :default-checked-keys="defaultCheckedKeys"
                            :default-expanded-keys="expandKeys"
                            show-checkbox
                            :expand-on-click-node="false"
                            @check="rightActive"
                        >
                        </vue-easy-tree>
                    </div>
                </div>
                <div class="btns">
                    <el-button :disabled="isLeft" type="primary" icon="el-icon-arrow-left" @click="handleToLeft"></el-button>
                    <el-button :disabled="isRight" type="primary" icon="el-icon-arrow-right" @click="handleToRight"></el-button>
                </div>
                <div class="treeRight borders">
                    <div class="title">{{ $t('selected') }}</div>
                    <div class="treeData" :style="{ width: '235px' }">
                        <vue-easy-tree
                            ref="rightTree"
                            class="virtutal-easy-tree checkbox-tree special-tree"
                            :props="defaultProps"
                            node-key="nodeId"
                            height="400px"
                            :data="rightCheckedList"
                            :render-content="renderContent"
                            default-expand-all
                            show-checkbox
                            :expand-on-click-node="false"
                            @check="leftActive"
                        >
                        </vue-easy-tree>
                    </div>
                </div>
            </div>
        </div>
    </BaseDialog>
</template>

<script>
import { cloneDeep } from 'lodash-es'
import BaseDialog from '@/components/Dialog'
import { cryptoNum } from '@/utils'
import VueEasyTree from '@wchbrad/vue-easy-tree'
import '@wchbrad/vue-easy-tree/src/assets/index.scss'

export default {
    props: {
        isDialog: { type: Boolean, default: false },
        list: { type: Array, default: () => [] },
        defaultKey: { type: Object, default: () => {} },
        isTime: { type: Boolean, default: false },
        isAuth: { type: Boolean, default: true },
    },
    components: { BaseDialog, VueEasyTree },
    data() {
        return {
            isLeft: true,
            isRight: true,
            isApply: false,
            defaultCheckedKeys: [],
            defaultProps: { label: 'label', children: 'children' },
            defaultTreeList: [],
            sourceList: cloneDeep(this.list),
            rightCheckedList: [],
            searchKey: '',
            expandKeys: [],
        }
    },
    created() {
        // this.disabledCheckbox(this.sourceList)
        this.setOptions(this.defaultKey)
        this.handleHasDefaultKey(this.defaultKey)
    },
    watch: {
        searchKey(nw) {
            this.filterTree(nw, this.rightCheckedList)
        },
    },
    mounted() {
        this.$nextTick(() => this.handleBackfill(this.defaultKey))
    },
    methods: {
        renderContent(h, { node }) {
            return h(
                'span',
                {
                    class: 'custom-tree-node-label custom-full-width',
                    attrs: { title: node.label },
                    style: { width: '70%' },
                },
                [h('span', { class: 'node-label', style: { display: 'block' } }, node.label)]
            )
        },
        filterTree(words, list) {
            const initList = cloneDeep(this.list)
            const { selectedList } = list[0]
            const flag = (selectedList && (selectedList[0] === 'any' || selectedList[0] === 'always')) || (!selectedList && (list[0]?.label === 'any' || list[0]?.label === 'always'))
            // 仅在左侧过滤操作，未添加至右侧
            if (flag) {
                this.handleTilterTree(initList)
            } else {
                //添加至右侧后，继续过滤操作
                let childs = [] // 存储所有子节点
                // 获取所有选中的父节点
                const pids = list.map((item) => {
                    const nids = item?.children.map((key) => {
                        return key.nodeId
                    })
                    childs = [...childs, ...nids]
                    return item.pid
                })
                initList.forEach((item) => {
                    if (pids.includes(item.pid)) {
                        const array = item.children.filter((key) => !childs.includes(key.nodeId))
                        item.children = array.filter((key) => key.label.includes(words))
                    } else {
                        item.children = item.children.filter((key) => key.label.includes(words))
                    }
                })
                this.sourceList = initList
                this.setDisabled(this.sourceList)
            }
        },
        handleTilterTree(list) {
            let array = [],
                treeList = []
            list.forEach((item) => {
                if (item.children && item.children.length) array = [...array, ...item.children]
            })
            const result = array.filter((key) => key.label.includes(this.searchKey))
            let faName = result.map((item) => {
                return item.prev_label
            })
            faName = [...new Set(faName)]
            faName.forEach((item) => {
                const children = result.filter((key) => key.prev_label == item)
                const current = list.find((el) => el.label === item)
                this.expandKeys.push(current.nodeId)
                current.children = children
                treeList.push(current)
            })
            if (!this.searchKey) {
                this.sourceList = cloneDeep(this.defaultTreeList)
                this.expandKeys = []
            } else {
                this.sourceList = treeList
            }
            this.setDisabled(this.sourceList)
        },
        // 若存在数据中不含有的选项，自动生成对应的选择项
        setOptions(data) {
            const result = []
            const names = data.selectedList?.map((item) => {
                return item.name ?? item
            })
            this.sourceList.forEach((item) => {
                if (item.children) {
                    item.children.forEach((key) => {
                        if (names.includes(key.label)) result.push(key.label)
                    })
                }
            })
            const resp = names.filter((key) => !result.includes(key))
            if (resp.length && !resp.includes('any') && !resp.includes('always')) {
                const itemArr = this.sourceList.find((item) => item.label == this.$t('userDefinedApply'))
                const appItem = {
                    isCheck: false,
                    label: resp[0],
                    value: resp[0],
                    nodeId: cryptoNum(),
                    pid: itemArr.pid,
                    prev_label: itemArr.label,
                    selectedAll: false,
                }
                itemArr.children.push(appItem)
            }
        },
        handleBackfill(data) {
            const tree = this.$refs.leftTree
            const sourecData = cloneDeep(this.sourceList)
            const names = data.selectedList?.map((item) => {
                return item.name ?? item
            })
            if (!names.includes('any') && !names.includes('always')) {
                const list = sourecData
                    .map((item) => {
                        item.children = item.children.filter((key) => names.includes(key.value))
                        return item
                    })
                    .filter((key) => key.children?.length)
                list.forEach((item) => {
                    item.children && item.children.forEach((key) => tree.remove(key.nodeId))
                })
                this.rightCheckedList = list
            }
            this.disabledCheckbox(this.sourceList)
        },
        /**
         * @description_按钮激活_向右穿梭
         * @param {*} node
         */
        rightActive(node) {
            if (node && node.children && !node.children.length) {
                this.isRight = !node.children.length
            } else {
                const tree = this.$refs.leftTree
                const checkedNodes = tree.getCheckedNodes()
                this.isRight = !checkedNodes.length
            }
            this.disabledCheckbox(this.sourceList)
        },
        handleToRight() {
            const tree = this.$refs.leftTree
            this.rightCheckedList = this.rightCheckedList.filter((key) => (this.defaultKey?.key === 'tr_list' ? key.label !== 'always' : key.label !== 'any'))
            const data = tree.getCheckedNodes(true) // 左侧tree选中的数据
            let selectedList = cloneDeep(data)
            const sourceData = cloneDeep(this.sourceList)
            let pids = [] // 存储对应的pid/删除左侧tree中的选中数据
            selectedList.forEach((item) => pids.push(item.pid) && tree.remove(item.nodeId))
            pids = [...new Set(pids)]
            const nodeList = sourceData.filter((key) => pids.includes(key.pid))
            nodeList.forEach((item) => (item.children = selectedList.filter((key) => key.pid === item.pid)))
            if (!this.rightCheckedList.length) {
                this.rightCheckedList = nodeList
            } else {
                const pids = nodeList.map((item) => {
                    return item.pid
                })
                pids.forEach((item) => {
                    const index = this.rightCheckedList.findIndex((key) => key.pid == item)
                    const node = nodeList.filter((el) => el.pid === item)
                    if (index > -1) {
                        const current = node[0]?.children
                        this.$set(this.rightCheckedList[index], 'children', [...this.rightCheckedList[index].children, ...current])
                    } else {
                        this.rightCheckedList = [...this.rightCheckedList, ...node]
                    }
                })
            }
            // tree.setCheckedKeys([])
            const eid = data?.[0]?.pid
            const expandId = sourceData.find((item) => item.pid === eid)?.nodeId
            if (expandId) this.expandKeys = [expandId]

            sourceData?.forEach((item) => tree.setChecked(item.nodeId, false))
            this.sourceList = Object.assign([], this.sourceList)
            this.sourceList.forEach((item) => (item.disabled = !(item.children && item.children.length)))
            this.isLeft = this.isRight = true
        },

        /**
         * @description_按钮激活_向左穿梭
         * @param {*} node
         */
        leftActive() {
            const tree = this.$refs.rightTree
            const checkedNodes = tree.getCheckedNodes()
            this.isLeft = !checkedNodes.length
        },
        handleToLeft() {
            const tree = this.$refs.rightTree
            const data = tree.getCheckedNodes(false, true) // 左侧tree选中的数据
            let selectedList = cloneDeep(data)
            const childNodes = selectedList.filter((item) => !item.children) // 过滤掉半选状态父节点
            childNodes.forEach((item) => tree.remove(item.nodeId)) // 删除左侧tree中的选中数据
            this.rightCheckedList = this.rightCheckedList.filter((key) => key.children && key.children.length)
            if (!this.rightCheckedList.length) this.rightCheckedList.push({ label: this.defaultKey?.key === 'tr_list' ? 'always' : 'any', disabled: true, nodeId: cryptoNum() })
            this.sourceList.forEach((item) => {
                const current = childNodes.filter((key) => key.pid === item.pid)
                current.length && this.$set(item, 'children', [...item.children, ...current])
            })
            const result = Object.assign([], this.sourceList)
            this.sourceList = result
            // tree.setCheckedKeys([])
            this.rightCheckedList?.forEach((item) => tree.setChecked(item.nodeId, false))
            this.isLeft = this.isRight = true
            this.disabledCheckbox(this.sourceList)
        },
        confirm() {
            let result = []
            this.rightCheckedList.forEach((item) => {
                const array = item?.children ?? []
                result = [...result, ...array]
            })
            if (!result.length) {
                const value = this.isTime ? 'always' : 'any'
                const item = { label: this.isTime ? 'always' : 'any', value }
                result.push(item)
            }
            if (result.length > 16) {
                this.$message.error(this.$t('cannotExceed16', { name: this.defaultKey.title }))
                return
            }
            if (result.length) {
                const list = result.map((item) => {
                    const obj = { name: item.value }
                    if (item.valueCn || item.valueEn) {
                        obj.valueCn = item.valueCn
                        obj.valueEn = item.valueEn
                    }
                    return obj
                })
                this.$emit('set-form', this.defaultKey.key, list)
            } else {
                this.beforeClose()
            }
        },
        beforeClose() {
            this.$emit('beforeClose')
        },
        changeCheck(item) {
            this.$set(item, 'isCheck', !item.isCheck)
        },
        disabledCheckbox(list, flag) {
            list.forEach((item) => (item.disabled = !(item.children && item.children.length)))
            if (flag) {
                this.handleTilterTree(list)
            } else {
                this.defaultTreeList = cloneDeep(list)
            }
        },
        setDisabled(list) {
            list.forEach((item) => (item.disabled = !!(item.children && !item.children?.length)))
        },
        // 是否存在默认值 any或always 并处理为右侧数据
        handleHasDefaultKey(obj) {
            if (obj?.selectedList?.includes('any') || obj?.selectedList?.includes('always')) {
                const item = obj?.selectedList.filter((key) => key == 'any' || key == 'always')
                this.rightCheckedList.push({ label: item[0], value: item[0], disabled: true, nodeId: cryptoNum() })
            }
        },
    },
}
</script>
