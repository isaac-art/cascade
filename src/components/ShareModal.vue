<template>
    <div class="share-modal-container" v-if="data.is_visible" @keyup="checkKey">
      <div class="share-modal">
        <button class="close-button" @click="data.is_visible = !data.is_visible">
          <i class="fa fa-window-close"></i>
        </button>
        <h1>Share Project</h1>
        <!-- <ul><li v-for="(user, index) in data.users" :key="index">{{ user }}</li></ul> -->
        <!-- invite, add an email address and it will send link to this project -->

        <button class="download-button" @click="$emit('download')">
            <i class="fa fa-download"></i> download project file
        </button>
      </div>
    </div>
  </template>
  
  <script>

  import moment from 'moment';
  export default {
    name: 'ShareModal',
    props: {
      data: Object
    },
    data() {
        return {}
    },
    methods: {
      checkKey(e){
        if(e.key == 'Escape'){
          this.data.is_visible = !this.data.is_visible;
        }
      }
    },
    components: {},
    created() {
      this.moment = moment;
    },
  }
  </script>
  
  <style scoped lang="scss">
  .close-button{
    position: absolute;
    top: 0;
    right: 0;
    background: none;
    border: none;
    font-size: 1.5em;
    color: #aaa;
    padding: 0.5em;
    cursor: pointer;
  }
  .share-modal{
    // position in the middle of the screen centered
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    width: 70vw;
    height: auto;
    min-height: 100px;
    min-width: 120px;
    display: flex;
    flex-direction: column;
    justify-content: left;
    align-items: left;
    z-index: 1000000000;
    background: #fff;
    border: none;
    border-radius: 12px;
    box-shadow: 0 0 10px rgba(0,0,0,0.2);
    padding: 20px;
    font: 1em sans-serif;
  }

  </style>
  