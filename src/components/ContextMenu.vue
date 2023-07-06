<template>
  <div class="context-menu" :style="menuStyle" v-if="is_visible">
    <div class="context-menu__item" 
        v-for="nodeType in nodeTypes" :key="nodeType" 
        @click="addNode(nodeType)">
      {{ nodeType }}
    </div>
  </div>
</template>

<script>
export default {
  props: ['is_visible', 'x', 'y', 'nodeTypes'],
  computed: {
    menuStyle() {
      return {
        left: `${this.x}px`,
        top: `${this.y}px`
      };
    }
  },
  methods: {
    addNode(nodeType) {
      this.$emit('add-node', nodeType);
    }
  }
}
</script>

<style scoped>
.context-menu {
    position: absolute;
    background-color: white;
    display: flex;
    flex-direction: column;
    justify-content: left;
    align-items: left;
    z-index: 1000000000;
    background: #fff;
    border: 1px solid red;
    border-radius: 12px;
    box-shadow: 0 0 10px rgba(0,0,0,0.2);
    padding: 20px;
    font: 1em sans-serif;
}

.context-menu__item {
    padding: 10px;
    cursor: pointer;
}
</style>
