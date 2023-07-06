<template>

<div class="prompt-node-container" :class="data.flag">
    <label class="prompt-node-label">PROMPT</label>

    <NodeToolbar :is-visible="selected" :position="toolbarPosition">
      <button @click="$emit('flag', id)"><i class="fa fa-flag"></i></button>
      <button @click="$emit('gen', id)"><i class="fa fa-arrow-down"></i></button><!--child-->
      <!-- <button @click="$emit('enhance', id)"><i class="fa fa-snowflake-o"></i></button>copy -->
      <button @click="$emit('fork', id)"><i class="fa fa-code-fork"></i></button><!--child-->
    </NodeToolbar>

    <div class="prompt-node">
        <textarea 
        v-model="data.prompt" 
        :placeholder="data.prompt" 
        class="nowheel nodrag"
        @input="changeRows"
        ></textarea>

        <!-- slider input from 0-1  -->
        <input type="range" min="0" max="1" step="0.01" 
            v-model="data.weight" class="nowheel nodrag" 
            >
    </div>

    <Handle type="target" :position="targetPosition" />
    <Handle type="source" :position="sourcePosition" />

</div>

</template>

<script>
import { Handle, Position } from '@vue-flow/core'
import { NodeToolbar } from '@vue-flow/node-toolbar'

export default {
    name: 'PromptNode',
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
.prompt-node-container{
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

.prompt-node-container:hover{
  box-shadow: 0 0 10px #ccc;
}
.prompt-node{
    width: 100%;
    height: 100%;
    padding: 20px;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
}

.prompt-node-label{
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
