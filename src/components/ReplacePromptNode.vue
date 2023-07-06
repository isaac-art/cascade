<template>

<div class="replace-prompt-node-container" :class="data.flag">
    <label class="replace-prompt-node-label">REPLACE</label>

    <NodeToolbar :is-visible="selected" :position="toolbarPosition">
        <button @click="$emit('flag', id)"><i class="fa fa-flag"></i></button>
        <button @click="$emit('enhance', id)"><i class="fa fa-snowflake-o"></i></button><!--copy-->
        <button @click="$emit('regen', id)"><i class="fa fa-refresh"></i></button><!--copy-->
        <button @click="$emit('fork', id)"><i class="fa fa-code-fork"></i></button><!--child-->
    </NodeToolbar>

    <div class="replace-prompt-node">
        <textarea 
        v-model="data.prompt" 
        :placeholder="data.prompt" 
        class="nowheel nodrag"
        @input="changeRows"
        ></textarea>
    </div>

    <Handle type="target" :position="targetPosition" />
    <Handle type="source" :position="sourcePosition" />

</div>

</template>

<script>
import { Handle, Position } from '@vue-flow/core'
import { NodeToolbar } from '@vue-flow/node-toolbar'

export default {
    name: 'ReplacePromptNode',
    props: {
        id: String,
        data: Object,
        selected: Boolean
    },
    data() {
        return {
            toolbarPosition: Position.Bottom,
            targetPosition: Position.Top,
            sourcePosition: Position.Bottom
        }
    },
    components: {
        NodeToolbar, Handle
    },
    methods: {
        changeRows(e){
            // set the height of the textarea to fit the content
            e.target.style.height = 'auto';
            e.target.style.height = e.target.scrollHeight + 'px';
        }
    }
}
</script>

<style scoped lang="scss">
.replace-prompt-node-container{
    background: #fff;
    border: 1px solid #eee;
    border-radius: 2px;
    min-height: 100px;
    min-width: 120px;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
}

.replace-prompt-node-container:hover{
    box-shadow: 0 0 10px #ccc;
}
.replace-prompt-node{
    width: 100%;
    height: 100%;
    padding: 20px;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
}

.replace-prompt-node-label{
    font: 8px sans-serif;
    position: absolute;
    top: 1px;
    left: 1px;
    padding: 5px;
    background: #000;
    color: #fff;
}

// textarea {
//     border: none;
// }
</style>
