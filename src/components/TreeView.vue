<template>

    <div class="tree-view">
        <template v-for="(nodeData, index) in data">
            <tree-node
                    :key="nodeData[nodeKeyProp]"
                    :keyProp="nodeKeyProp"
                    :renameOnDblClick="renameNodeOnDblClick"
                    :childrenProp="nodeChildrenProp"
                    :labelProp="nodeLabelProp"
                    :data="nodeData"
                    :draggable="false"
                    :defaultIconClass="defaultIconClass"
                    :iconClassProp="iconClassProp"
                    :showIcon="showIcons"
                    :prependIconClass="prependIconClass"
                    :contextMenu="false"
                    ref="rootNodes"
                    @nodeSelect="nodeSelect"
                    @nodeDragStart="nodeDragStart"
                    @deleteNode="deleteNode">
            </tree-node>
        </template>

    </div>

</template>

<script>

    import TreeNode from './TreeNode.vue';
    import EventBus from '../EventBus.js';
    import DropBetweenZone from './DropBetweenZone.vue'

    export default {
        props: {
            data: {
                type: Array,
                required: true
            },
            allowMultiple: {
                type: Boolean,
                default: false
            },
            nodeKeyProp: {
                type: String,
                default: 'id'
            },
            nodeChildrenProp: {
                type: String,
                default: 'children'
            },
            nodeLabelProp: {
                type: String,
                default: 'name'
            },
            nodesDraggable: {
                type: Boolean,
                default: false
            },
            contextMenu: {
                type: Boolean,
                default: true
            },
            contextMenuItems: {
                type: Array,
                default: [{code: 'DELETE_NODE', label: 'Delete node'}, {code: 'RENAME_NODE', label: 'Rename node'}]
            },
            renameNodeOnDblClick: {
                type: Boolean,
                default: true
            },
            // class added to every icon no matter what
            prependIconClass: {
                type: String,
                default: null
            },
            // default icon if node icon is not specified
            defaultIconClass: {
                type: String,
                default: null
            },
            // where to search for node icon
            iconClassProp: {
                type: String,
                default: "icon"
            },
            // show icons
            showIcons: {
                type: Boolean,
                default: false
            }
        },
        data() {
            return {
                draggedNode: null
            }
        },
        components: {
            TreeNode,
            DropBetweenZone,
        },
        methods: {
            createNodeMap() {
                if (this.nodeMap === undefined) {
                    let nodeMap = this.nodeMap = new Map()

                    let nodes = this.$refs.rootNodes.slice()

                    for (let i = 0; i < nodes.length; i++) {
                        let tmpNode = nodes[i]
                        nodes.push(...tmpNode.getChildNodes())
                    }

                    for (let tmpNode of nodes) {
                        nodeMap.set(tmpNode.data[this.nodeKeyProp], tmpNode) // TODO: change to getter
                    }
                }
            },
            getNodeByKey(key) {
                return this.nodeMap.get(key)
            },
            // event bubbles up to the roots
            nodeSelect(node, isSelected) {
                this.$emit('nodeSelect', node, isSelected)
                if (isSelected) {
                    if (this.selectedNode !== null) {
                        this.selectedNode.deselect()
                    }
                    this.selectedNode = node
                } else if (node === this.selectedNode) {
                    this.selectedNode = null
                }
            },
            nodeDragStart() {
                EventBus.$on('dropOK', this.cutNode)
            },
            cutNode() {
                EventBus.$off('dropOK')
                let idx = this.data.indexOf(window._bTreeView.draggedNodeData)
                this.data.splice(idx, 1)
                // let's notify that node data was successfully cut (removed from array)
                EventBus.$emit('cutOK')
            },
            draggingStarted(draggedNode) {
                this.draggedNode = draggedNode
                // let's listen for the drag end event
                EventBus.$on('nodeDragEnd', this.draggingEnded)
            },
            draggingEnded() {
                // stop listening for the dragging end event
                EventBus.$off('nodeDragEnd', this.draggingEnded)
                this.draggedNode = null
            },
            dropNodeAtPosition(pos) {
                // position can change if we move node within the same parent node (same level)
                // so it's better to remember node at previous position
                let insertAfter = pos - 1 < 0 ? null : this.data[pos - 1]
                EventBus.$on('cutOK', () => {
                    let pos = this.data.indexOf(insertAfter) + 1
                    this.data.splice(pos, 0, window._bTreeView.draggedNodeData)
                    delete window._bTreeView.draggedNodeKey
                    delete window._bTreeView.draggedNodeData
                    EventBus.$off('cutOK');
                })
                EventBus.$emit('dropOK')
            },
            deleteNode(nodeData) {
                let nodes = this.data
                let idx = nodes.indexOf(nodeData)
                nodes.splice(idx, 1)
            },
            menuItemSelected(item, node) {
                switch (item.code) {
                    case 'DELETE_NODE':
                        node.delete()
                    case 'RENAME_NODE':
                        node.startRenaming()
                    default:
                        this.$emit('contextMenuItemSelect', item, node)
                }
            }
        },
        created() {
            this.selectedNode = null
            EventBus.$on('nodeDragStart', this.draggingStarted)
            EventBus.$on('contextMenuItemSelect', this.menuItemSelected)
            this.$nextTick(() => {
                this.createNodeMap()
            })
        }
    }

</script>

<style>

    .tree-view {
        text-align: left;
    }

</style>
