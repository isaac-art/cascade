<template>
  <div class="image-node-container" :class="data.flag">

    <label class="image-node-label">IMAGE</label>

    <NodeToolbar :is-visible="selected" :position="toolbarPosition">
      <button :v-model="data.flag" @click="$emit('flag', id)" :class="data.flag"><i class="fa fa-flag"></i></button>
      <button @click="$emit('regen', id)"><i class="fa fa-refresh"></i></button><!--copy-->
      <!-- <button @click="$emit('enhance', id)"><i class="fa fa-snowflake-o"></i></button>enhance -->
      <!-- <button @click="$emit('add', id)"><i class="fa fa-pencil"></i></button>add -->
      <!-- <button @click="$emit('replace', id)"><i class="fa fa-eraser"></i></button>replace -->
    </NodeToolbar>

    <div class="image-node">
      <img :src="data.src" :alt="data.alt" v-if="data.src"/>

      <i class="regen-button fa fa-refresh fa-3x" 
        :class="{'fa-spin': data.generating}"     
        v-else @click="triggerRegen"></i>
    </div>

    <Handle type="target" :position="targetPosition" />
    <Handle type="source" :position="sourcePosition" />
  </div>
</template>

<script>
import { Handle, Position } from '@vue-flow/core'
import { NodeToolbar } from '@vue-flow/node-toolbar'

export default {
  name: 'ImageNode',
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
  methods:{
    triggerRegen(){
      // add fa-spin class to the regen-button to make it spin
      this.$el.querySelector('.regen-button').classList.add('fa-spin')
      this.$emit('regen', this.id)
    }
  }
}
</script>

<style scoped lang="scss">
.regen-button{
  cursor: pointer;
  color: #ccc
}
.regen-button:hover{
  color: #000;
}

.image-node-container{
    background: #fff;
    border: 1px solid #eee;
    border-radius: 2px;
    min-height: 300px;
    min-width: 300px;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
}

.image-node-container:hover{
  box-shadow: 0 0 10px #ccc;
}

.image-node{
    width: 100%;
    height: 100%;
    padding: 20px;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
}

.image-node-label{
    font: 8px sans-serif;
    position: absolute;
    top: 1px;
    left: 1px;
    padding: 5px;
    background: #000;
    color: #fff;
}

img {
  max-width: 300px;
}




</style>
