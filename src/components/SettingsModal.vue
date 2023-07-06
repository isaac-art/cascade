<template>
    <div class="settings-modal-container" v-if="data.is_visible" @keyup="checkKey">
      <div class="settings-modal">
        <button class="close-button" @click="data.is_visible = !data.is_visible">
          <i class="fa fa-window-close"></i>
        </button>
        <h1>Project Title</h1>
        <input type="text" v-model="data.project_name" placeholder="project name">
        
        <!-- Project Created: {{ moment(data.created_at).format('LLLL') }}  -->
        <h1>Dreamstudio API KEY</h1> 
        <input type="text" v-model="data.api_key" placeholder="apikey">
        <a href="https://platform.stability.ai/rest-api" target="_blank">stability.ai api</a>
      </div>
    </div>
  </template>
  
  <script>

  import moment from 'moment';
  export default {
    name: 'SettingsModal',
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
  .settings-modal{
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


    h1 {
      font-size: 1em;
      color: #333;
      margin-bottom: 15px;
    }

    // add style for input
    input[type="text"] {
      width: 95%;
      padding: 10px;
      margin-bottom: 15px;
      font-size: 1em;
      border-radius: 5px;
      border: 1px solid #ccc;
    }

    a {
      color: #7e2956; // Change to your preferred color
      text-decoration: none; // Removes the default underline from links

      &:hover { // Styles for when the link is hovered over
        text-decoration: underline; // Adds an underline when hovered over
        color: #c91b78; // Changes color when hovered over
      }

      &:active { // Styles for when the link is actively being clicked
        color: #9d0b59; // Change to your preferred color
      }
    }
  }

  </style>
  