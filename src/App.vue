<template>
  <div id="app">
    <VueFlow 
      v-model="elements" 
      @drop="dropImage"
      :default-zoom="1"
      :max-zoom="2"
      :min-zoom="0.5"
      :snap-to-grid="true"
      :snap-grid=[20,20]
      :auto-connect="connector"
      fit-view-on-init
      :zoom-on-double-click="false"
      @pane-context-menu="openContextMenu"
      @pane-click="hideAllModal"
    >
    <template #node-prompt="props">
      <PromptNode v-bind="props" 
        @flag="flagWithChildren" @enhance="enhancePrompt" @fork="forkNode" @gen="makeImageAndGen"/>
    </template>
    <template #node-negativeprompt="props">
      <NegativePromptNode v-bind="props" 
        @flag="flagWithChildren" @enhance="enhancePrompt" @fork="forkNode" @gen="makeImageAndGen"/>
    </template>
    <template #node-replace="props">
      <ReplacePromptNode v-bind="props" 
        @flag="flagWithChildren" @enhance="enhancePrompt" @fork="forkNode" />
    </template>
    <template #node-image="props">
      <ImageNode v-bind="props" 
        @flag="flagWithChildren" @enhance="enhanceImage" @add="imageAddNode" 
        @replace="imageReplaceNode" @regen="regenNode"/>
    </template>

      <Background/>
      <Controls>
        <ControlButton @click="downloadProject"><i class="fa fa-download" aria-atomic="Download"></i></ControlButton>
        <ControlButton @click="uploadProject"><i class="fa fa-upload"></i></ControlButton>
        <!-- <ControlButton @click="undo"><i class="fa fa-undo"></i></ControlButton> -->
        <ControlButton @click="toggleSettingsModal"><i class="fa fa-cogs"></i></ControlButton>
        <!-- <ControlButton @click="toggleShareModal"><i class="fa fa-share"></i></ControlButton> -->
      </Controls>
    </VueFlow>
    <SettingsModal :data="settings"/>
    <ShareModal :data="share" @download="downloadProject"/>
    <ContextMenu :is_visible="contextMenu.is_visible" 
      :x="contextMenu.x" :y="contextMenu.y" 
      :nodeTypes="contextNodeTypes" @add-node="addNode"/>
  </div>
</template>

<script>
import axios from 'axios'
import { markRaw, ref } from 'vue'
import { VueFlow  } from '@vue-flow/core'
import { Background } from '@vue-flow/background'
import { Controls, ControlButton } from '@vue-flow/controls'
import ImageNode from './components/ImageNode.vue'
import PromptNode from './components/PromptNode.vue'
import NegativePromptNode from './components/NegativePromptNode.vue'
import SettingsModal from './components/SettingsModal.vue'
import ShareModal from './components/ShareModal.vue'
import ContextMenu from './components/ContextMenu.vue'
import ReplacePromptNode from './components/ReplacePromptNode.vue'


const flags = ['', 'red-flag', 'blue-flag', 'green-flag', 'yellow-flag', 'orange-flag', 'purple-flag', 'pink-flag', 'brown-flag', 'black-flag', 'white-flag'];


export default {
  name: 'App',
  components: {
    VueFlow, ImageNode, PromptNode, Controls, 
    Background, ControlButton, SettingsModal,
    ShareModal, ContextMenu, ReplacePromptNode,
    NegativePromptNode
  },
  data() {
    return {
      nodeTypes: {
        image: markRaw(ImageNode),
        prompt: markRaw(PromptNode),
        replace: markRaw(ReplacePromptNode),
        negativeprompt: markRaw(NegativePromptNode),
      },
      contextMenu: {
        is_visible: false,
        x: 0,
        y: 0,
      },
      contextNodeTypes: ['image', 'prompt', 'negativeprompt'],
      settings:{
        engine_id: "stable-diffusion-xl-beta-v2-2-2",
        is_visible: false,
        api_key: '',
        project_name: 'My Project',
        created_at: new Date(),
        updated_at: new Date(),
      },
      share:{
        users: [{email:'123@demo.com',active:false},{email:'1321@demo.com',active:false}],
        use_password: false, password: '', is_visible: false,
      },
      elements: [
      ]
    }
  },
  computed: {
    countNodes(){ return this.elements.filter(el => !el.source).length },
    countEdges(){ return this.elements.filter(el => el.source).length },
    countElements(){ return this.elements.length },
    largestNodeID() {
      const nodes = this.elements.filter(el => !el.source)
      if (nodes.length === 0) { return 0 }
      const ids = nodes.map(el => parseInt(el.id))
      return Math.max(...ids)
    }
  },
  mounted(){
    this.loadLocalStorage()
    // setInterval(this.saveLocalStorage, 10000)
    // if url has ?key=....then set the api key to this value
    var root = this;
    // load the default.json file 
    axios.get('default.json').then(response => {
      this.elements = response.data.elements
      this.settings = response.data.settings
      this.share = response.data.share
    }).then(function (res){

      const urlParams = new URLSearchParams(window.location.search);
      var key = urlParams.get('key');
      if(key){ 
        root.settings.api_key = key; 
        console.log("set api key", root.settings.api_key) 
      }

      // show the settings modal on launch
      if(root.settings.api_key === ''){
        root.settings.is_visible = true;
      }
      if(root.settings.project_name === ''){
        root.settings.project_name = root.makeProjectTitle()
      }
      root.settings.created_at = new Date()
      root.settings.updated_at = new Date()
      
    })
    .catch(function (error) {
      console.log(error);
    });

  },
  created() {
    window.addEventListener('dragover', (e) => { e.preventDefault(); }, false);
    window.addEventListener('drop', (e) => { e.preventDefault(); }, false);
  },
  methods: {
    dropImage(e){
      e.preventDefault()
      console.log("dropping ")
      // Check if any files were dropped
      if (e.dataTransfer.items) {
        for (let i = 0; i < e.dataTransfer.items.length; i++) {
          // If the dropped items are files, handle them
          if (e.dataTransfer.items[i].kind === 'file') {
            let file = e.dataTransfer.items[i].getAsFile();
            
            // Check if the file is an image
            if (file.type.startsWith('image/')) {
              let pos = { x: e.clientX, y: e.clientY }
              this.contextMenu.is_visible = false;
              // unselect any other node
              this.elements.forEach(el => { el.selected = false })
              
              // add new node
              let reader = new FileReader();
              reader.onloadend = () => {
                let img = new Image();
                img.onload = () => {
                  let canvas = document.createElement('canvas');
                  let ctx = canvas.getContext('2d');
                  canvas.width = Math.round(img.width / 64) * 64; // make it multiple of 64
                  canvas.height = Math.round(img.height / 64) * 64; // make it multiple of 64
                  ctx.drawImage(img, 0, 0, canvas.width, canvas.height);

                  let base64 = canvas.toDataURL(); // this is the base64 encoded and resized image

                  this.elements.push({
                    id: `${this.largestNodeID+1}`,
                    label: `image ${this.largestNodeID+1}`,
                    type: 'image',
                    position: pos,
                    data: {"src": base64 , "alt": 'uploaded file'},
                    selected: true
                  })
                };
                img.src = reader.result;
              }
              reader.readAsDataURL(file);
            }
          }
        }
      }

    },
    makeProjectTitle(){
      var words = [
        'cat', 'summer', 'french', 'powder', 'whiskey', 'sunset', 
        'ocean', 'autumn', 'never', 'soon', 'dreamboat', 'fun',
        'laughter', 'mountain', 'candle', 'breeze', 'delicious', 
        'moonlight', 'chocolate', 'paradise', 'harmony', 'adventure', 
        'serenade', 'jubilant', 'melody', 'blissful', 'serendipity', 
        'enchanted', 'wanderlust', 'sparkle', 'crimson', 'fireworks', 
        'velvet', 'giggles', 'pizza', 'sugar', 'sunshine', 'rainbow']
      var select_three = []
      for (let i = 0; i < 3; i++) {
        var index = Math.floor(Math.random() * words.length);
        select_three.push(words[index])
      }
      return select_three.join('-')
    },
    connector(params){
      console.log(params) // {source:num, target:num}
      if (params.source === params.target) {return false } 
      let isAnimated = false;
      let target_type = this.elements.find(el => el.id === params.target).type
      let source_type = this.elements.find(el => el.id === params.source).type
      // replace node can only be connected to a source image node
      if (target_type === 'replace' && source_type !== 'image' ||
          target_type === 'replace' && source_type === 'image') {
        console.log("replace node can only be connected to a source image node")
        return false
      }

      // other checks based on rules
      console.log("making edge")
      let edge =  {
        id: `edge-${params.source}-${params.target}`,
        source: params.source,
        target: params.target,
        // label: `Edge ${params.source}-${params.target}`,
        animated: isAnimated
      }
      console.log(edge)
      return edge
    }
    ,
    addNode(nodeType){
      console.log("adding node", nodeType)
      let pos = { x: this.contextMenu.x, y: this.contextMenu.y }
      this.contextMenu.is_visible = false;
      // unselect any other node
      this.elements.forEach(el => { el.selected = false })
      this.elements.push({
        id: `${this.largestNodeID+1}`,
        label: `${nodeType} ${this.largestNodeID+1}`,
        type: nodeType,
        position: pos,
        data: {"flag": ""},
        selected: true
      })
    },
    openContextMenu(event){
      event.preventDefault();
      console.log(event)
      // return
      this.contextMenu.x = event.x;
      this.contextMenu.y = event.y;
      this.contextMenu.is_visible = true;
    },
    hideAllModal(){
      this.contextMenu.is_visible = false;
      this.settings.is_visible = false;
      this.share.is_visible = false;
    },
    saveLocalStorage(){
      console.log("saving...")
      //deep copy this.elements
      // var save_data = JSON.parse(JSON.stringify(this.elements))
      // // remove src from image nodes
      // save_data.forEach(el => {
      //   if(el.type === 'image'){
      //     el.data.src = false
      //   }
      // })
      // localStorage.setItem('flow-data', JSON.stringify(save_data))
      localStorage.setItem('flow-settings', JSON.stringify(this.settings))
      // localStorage.setItem('flow-share', JSON.stringify(this.share))
    },
    loadLocalStorage(){
      // const data = localStorage.getItem('flow-data')
      const settings = localStorage.getItem('flow-settings')
      // const share = localStorage.getItem('flow-share')
      // if (data) {  this.elements = JSON.parse(data) }
      if (settings) { this.settings = JSON.parse(settings)  }
      // if (share) {  this.share = JSON.parse(share) }
    },
    downloadProject(){
      this.saveLocalStorage()
      const project = { settings: this.settings, elements: this.elements, share: this.share }
      const dataStr = "data:text/json;charset=utf-8," + encodeURIComponent(JSON.stringify(project));
      const downloadAnchorNode = document.createElement('a');
      downloadAnchorNode.setAttribute("href",     dataStr);
      downloadAnchorNode.setAttribute("download", `${this.settings.project_name}.json`);
      document.body.appendChild(downloadAnchorNode); // required for firefox
      downloadAnchorNode.click();
      downloadAnchorNode.remove();
    },
    uploadProject(){
      // open file dialog, accept .json files
      const input = document.createElement('input');
      input.type = 'file';
      input.accept = '.json';
      input.onchange = e => { 
        // getting a hold of the file reference
        const file = e.target.files[0]; 
        // setting up the reader
        const reader = new FileReader();
        reader.readAsText(file,'UTF-8');
        // here we tell the reader what to do when it's done reading...
        reader.onload = readerEvent => {
          const content = readerEvent.target.result; // this is the content!
          const project = JSON.parse(content);
          this.elements = project.elements;
          this.settings = project.settings;
          this.share = project.share;
        }
      }
      input.click();
    },
    undo(){},
    toggleSettingsModal(){
      this.settings.is_visible = !this.settings.is_visible
      if (this.settings.is_visible) {
        this.share.is_visible = false;
      }
    },
    toggleShareModal(){
      this.share.is_visible = !this.share.is_visible
      if (this.share.is_visible) {
        this.settings.is_visible = false;
      }
    },
    // NODE EMITTERS
    forkNode(source_node_id){
      console.log("forking node", source_node_id)
      const source_node = this.elements.find(el => el.id === source_node_id)
      const new_node = JSON.parse(JSON.stringify(source_node))
      new_node.id = `${this.largestNodeID+1}`
      new_node.position.x += 150
      new_node.position.y += 150
      this.elements.push(new_node)
      this.elements.push({
        id: `edge-${source_node.id}-${new_node.id}`,
        source: source_node.id,
        target: new_node.id,
        animated: false
      })
      // unselect source_node and select new_node
      source_node.selected = false;
      new_node.selected = true;
    },
    regenNode(source_node_id){
      console.log("regen node")
      console.log("default is text to image")
      if(this.settings.engine_id === undefined){this.settings.engine_id = "stable-diffusion-xl-beta-v2-2-2"}
      
      // get the parent nodes, where this node is target of edge
      let node = this.elements.find(el => el.id === source_node_id)
      // remove previous image from target node, use default alt
      node.data.generating = true
      node.data.src = false
      node.data.alt = this.settings.project_name

      let edges = this.elements.filter(el => el.target === source_node_id)
      let source_nodes = edges.map(el => this.elements.find(e => e.id === el.source))
      let prompt_nodes = source_nodes.filter(el => el.type === 'prompt')
      let negative_prompt_nodes = source_nodes.filter(el => el.type === 'negativeprompt')
      let image_nodes = source_nodes.filter(el => el.type === 'image')
      // let replace_nodes = source_nodes.filter(el => el.type === 'replace')
      // let style_nodes = source_nodes.filter(el => el.type === 'style')
      // let style_preset = false// 3d-model analog-film anime cinematic comic-book digital-art enhance fantasy-art isometric line-art low-poly modeling-compound neon-punk origami photographic pixel-art tile-texture

      let prompts = []
      for (let i = 0; i < prompt_nodes.length; i++) {
        var p = {}
        p.text = prompt_nodes[i].data.prompt
        if(prompt_nodes[i].data.weight == undefined){
          p.weight = 0.5
        }else{
          p.weight = prompt_nodes[i].data.weight * 1
        }
        prompts.push(p)
      }
      for (let i = 0; i < negative_prompt_nodes.length; i++) {
        var p = {}
        p.text = negative_prompt_nodes[i].data.prompt
        if(negative_prompt_nodes[i].data.weight == undefined){
          p.weight = -0.5
        }else{
          p.weight = negative_prompt_nodes[i].data.weight * -1
        }
        prompts.push(p)
      }

      var jsondata = {
        height: 512, width: 512, text_prompts: prompts,
        cfg_scale: 7, clip_guidance_preset: "NONE", //FAST_BLUE FAST_GREEN NONE SIMPLE SLOW SLOWER SLOWEST
        samples: 1, steps: 30,
      }

      var headers = { 
        'Authorization': 'Bearer '+this.settings.api_key,
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        'Access-Control-Allow-Origin': '*',
      };
      console.log("text-to-image payload", data)
      var data = null;
      var url  = "https://api.stability.ai/v1/generation/"+this.settings.engine_id+"/text-to-image"
      if(image_nodes.length > 0){
        // use image-to-image
        headers['Content-Type'] = 'multipart/form-data';
        url  = "https://api.stability.ai/v1/generation/"+this.settings.engine_id+"/image-to-image"
        var init_image = image_nodes[0].data.src //use first image node, we can only take 1
        // init_image is base64 encoded image
        // "invalid mime type for init_image: text/plain; charset=utf-8 is not image/jpeg, image/png, image/gif, or image/webp"
        // to fix 
        var base64ImageContent = init_image.replace(/^data:(.*);base64,/, "");
        var mime = 'image/png'; // Here, we specify the image MIME type as PNG
        var init_image_blob = base64ToBlob(base64ImageContent, mime);

        function base64ToBlob(base64, mime) {
            mime = mime || '';
            var sliceSize = 1024;
            var byteChars = window.atob(base64);
            var byteArrays = [];
            for (var offset = 0, len = byteChars.length; offset < len; offset += sliceSize) {
                var slice = byteChars.slice(offset, offset + sliceSize);
                var byteNumbers = new Array(slice.length);
                for (var i = 0; i < slice.length; i++) {
                    byteNumbers[i] = slice.charCodeAt(i);
                }
                var byteArray = new Uint8Array(byteNumbers);
                byteArrays.push(byteArray);
            }
            return new Blob(byteArrays, {type: mime});
        }

        var formData = new FormData();
        for(var i = 0; i < prompts.length; i++){
          formData.append(`text_prompts[${i}][text]`, prompts[i].text);
          formData.append(`text_prompts[${i}][weight]`, prompts[i].weight);
        }
        formData.append('cfg_scale', 7);
        formData.append('clip_guidance_preset', "NONE");
        formData.append('samples', 1);
        formData.append('steps', 30);
        formData.append('init_image', init_image_blob);
        formData.append('init_image_mode', 'IMAGE_STRENGTH');
        formData.append('image_strength', 0.35);
        data = formData
      }else{
        // use text-to-image
        data = jsondata
      }
      
      axios.post(url, data, {headers: headers}).then(response => {
        response.data.artifacts.forEach((artifact) => {
          if(artifact.finishReason == 'SUCCESS'){
            // make base64 image into a src url by prepending the data type
            var url = 'data:image/png;base64,'+artifact.base64
            node.data.src = url
            node.data.alt = `${this.settings.project_name}-${node.id}`
          }
        })
      }).catch(function (error) {
        console.log(error);
      }).finally(function(){
        node.data.generating = false
      });
    
    },
    makeImageAndGen(source_node_id){
      // add image node, add edge from this node to image node
      // trigger regen on image node
      console.log("make image and gen")
      let source_node = this.elements.find(el => el.id === source_node_id)
      let image_node = {
        id: `${this.largestNodeID+1}`,
        label: `image ${this.largestNodeID+1}`,
        type: 'image',
        position: {x: source_node.position.x , y: source_node.position.y + 250},
        data: {src: false, alt: this.settings.project_name, generating: true},
        selected: true
      }
      source_node.selected = false;
      this.elements.push(image_node)
      this.elements.push({
        id: `edge-${source_node.id}-${image_node.id}`,
        source: source_node.id,
        target: image_node.id,
        animated: false
      })
      this.regenNode(image_node.id)
    },
    enhancePrompt(source_node_id){console.log("enhance prompt")},
    enhanceImage(source_node_id){console.log("enhance image")},
    imageAddNode(source_node_id){console.log("add node")},
    imageReplaceNode(source_node_id){console.log("replace node")},
    flagCycle(node){
      let index = flags.indexOf(node.data.flag);
      if (index === flags.length - 1){ index = 0; } else { index++; }
      node.data.flag = flags[index];
    },
    flagWithChildren(source_node_id){
      const node = this.elements.find(el => el.id === source_node_id);
      this.flagCycle(node);
      const flag_idx = flags.indexOf(node.data.flag);
      const edges = this.elements.filter(el => el.source === source_node_id);
      const children = edges.map(el => this.elements.find(e => e.id === el.target));
      
      children.forEach(child => { 
        child.data.flag = flags[flag_idx];
      }) 
    }
  }

}
</script>

<style lang="scss">
body{
  margin: 0;
  padding: 0;
}

#app{
  width: 100vw;
  height: 100vh;
}

.vue-flow {
    position: relative;
    width: 100vw;
    height: 100vh;
    overflow: hidden;
    z-index: 0;
}

.vue-flow__container {
    position: absolute;
    height: 100%;
    width: 100%;
    left: 0;
    top: 0;
}

.vue-flow__pane {
z-index: 1;
}

.vue-flow__pane.draggable {
    cursor: grab;
    }

.vue-flow__pane.dragging {
    cursor: grabbing;
    }

.vue-flow__pane.selection {
    cursor: pointer;
    }

.vue-flow__transformationpane {
transform-origin: 0 0;
z-index: 2;
pointer-events: none;
}

.vue-flow__viewport {
z-index: 4;
}

.vue-flow__selection {
z-index: 6;
}

.vue-flow__edge-labels {
position: absolute;
width: 100%;
height: 100%;
pointer-events: none;
-webkit-user-select: none;
    -moz-user-select: none;
        user-select: none;
}

.vue-flow__nodesselection-rect:focus,
.vue-flow__nodesselection-rect:focus-visible {
outline: none;
}

.vue-flow .vue-flow__edges {
pointer-events: none;
overflow: visible;
}

.vue-flow__edge-path,
.vue-flow__connection-path {
stroke: #b1b1b7;
stroke-width: 2;
fill: none;
}

.vue-flow__edge {
pointer-events: visibleStroke;
cursor: pointer;
}

.vue-flow__edge.animated path {
    stroke-dasharray: 5;
    animation: dashdraw 0.5s linear infinite;
    }

.vue-flow__edge.animated path.vue-flow__edge-interaction {
    stroke-dasharray: none;
    animation: none;
    }

.vue-flow__edge.inactive {
    pointer-events: none;
    }

.vue-flow__edge.selected,
.vue-flow__edge:focus,
.vue-flow__edge:focus-visible {
    outline: none;
    }

.vue-flow__edge.selected .vue-flow__edge-path,
.vue-flow__edge:focus .vue-flow__edge-path,
.vue-flow__edge:focus-visible .vue-flow__edge-path {
    stroke: #555;
    }

.vue-flow__edge-textwrapper {
    pointer-events: all;
    }

.vue-flow__edge-textbg {
    fill: white;
    }

.vue-flow__edge-text {
    pointer-events: none;
    -webkit-user-select: none;
        -moz-user-select: none;
            user-select: none;
}

.vue-flow__connection {
pointer-events: none;
}

.vue-flow__connection .animated {
    stroke-dasharray: 5;
    animation: dashdraw 0.5s linear infinite;
    }

.vue-flow__connectionline {
z-index: 1001;
}

.vue-flow__nodes {
pointer-events: none;
transform-origin: 0 0;
}

.vue-flow__node-default,
.vue-flow__node-input,
.vue-flow__node-output {
border-width: 1px;
border-style: solid;
border-color: #bbb;
}

.vue-flow__node-default.selected,
.vue-flow__node-default:focus,
.vue-flow__node-default:focus-visible,
.vue-flow__node-input.selected,
.vue-flow__node-input:focus,
.vue-flow__node-input:focus-visible,
.vue-flow__node-output.selected,
.vue-flow__node-output:focus,
.vue-flow__node-output:focus-visible {
    outline: none;
    border: 1px solid #555;
    }

.vue-flow__node {
position: absolute;
-webkit-user-select: none;
    -moz-user-select: none;
        user-select: none;
pointer-events: all;
transform-origin: 0 0;
box-sizing: border-box;
cursor: grab;
}

.vue-flow__node.dragging {
    cursor: grabbing;
    }

.vue-flow__nodesselection {
z-index: 3;
transform-origin: left top;
pointer-events: none;
}

.vue-flow__nodesselection-rect {
    position: absolute;
    pointer-events: all;
    cursor: grab;
    }

.vue-flow__nodesselection-rect.dragging {
        cursor: grabbing;
        }

.vue-flow__handle {
position: absolute;
pointer-events: none;
min-width: 5px;
min-height: 5px;
}

.vue-flow__handle.connectable {
    pointer-events: all;
    cursor: crosshair;
    }

.vue-flow__handle-bottom {
    top: auto;
    left: 50%;
    bottom: -4px;
    transform: translate(-50%, 0);
    }

.vue-flow__handle-top {
    left: 50%;
    top: -4px;
    transform: translate(-50%, 0);
    }

.vue-flow__handle-left {
    top: 50%;
    left: -4px;
    transform: translate(0, -50%);
    }

.vue-flow__handle-right {
    right: -4px;
    top: 50%;
    transform: translate(0, -50%);
    }

.vue-flow__edgeupdater {
cursor: move;
pointer-events: all;
}

.vue-flow__panel {
position: absolute;
z-index: 5;
margin: 15px;
}

.vue-flow__panel.top {
    top: 0;
    }

.vue-flow__panel.bottom {
    bottom: 0;
    }

.vue-flow__panel.left {
    left: 0;
    }

.vue-flow__panel.right {
    right: 0;
    }

.vue-flow__panel.center {
    left: 50%;
    transform: translateX(-50%);
    }

@keyframes dashdraw {
from {
    stroke-dashoffset: 10;
}
}

:root {
  --vf-node-bg: #fff;
  --vf-node-text: #222;
  --vf-connection-path:  #b1b1b7;
  --vf-handle: #555;
}

.vue-flow__edge.updating .vue-flow__edge-path {
    stroke: #777;
    }

.vue-flow__edge-text {
font-size: 10px;
}

.vue-flow__edge-textbg {
fill: #fff;
}

.vue-flow__connection-path {
stroke: var(--vf-connection-path);
}

.vue-flow__node {
cursor: grab;
}

.vue-flow__node.selectable:focus,
.vue-flow__node.selectable:focus-visible {
    outline: none;
    }

.vue-flow__node-default,
.vue-flow__node-input,
.vue-flow__node-output {
  padding: 10px;
  border-radius: 3px;
  width: 150px;
  font-size: 12px;
  text-align: center;
  border-width: 1px;
  border-style: solid;
  color: var(--vf-node-text);
  background-color: var(--vf-node-bg);
  border-color: var(--vf-node-color);
}

.vue-flow__node-default.selected,
.vue-flow__node-default.selected:hover,
.vue-flow__node-input.selected,
.vue-flow__node-input.selected:hover,
.vue-flow__node-output.selected,
.vue-flow__node-output.selected:hover {
    box-shadow: 0 0 0 0.5px var(--vf-box-shadow);
}

.vue-flow__node-default .vue-flow__handle, .vue-flow__node-input .vue-flow__handle, .vue-flow__node-output .vue-flow__handle {
    background: var(--vf-handle);
}

.vue-flow__node-default.selectable:hover, .vue-flow__node-input.selectable:hover, .vue-flow__node-output.selectable:hover {
    box-shadow: 0 1px 4px 1px rgba(0, 0, 0, 0.08);
}

.vue-flow__node-input {
  --vf-node-color: var(--vf-node-color, #0041d0);
  --vf-handle: var(--vf-node-color, #0041d0);
  --vf-box-shadow: var(--vf-node-color, #0041d0);

  background: var(--vf-node-bg);
  border-color: var(--vf-node-color, #0041d0);
}

.vue-flow__node-input.selected,
.vue-flow__node-input:focus,
.vue-flow__node-input:focus-visible {
    outline: none;
    border: 1px solid var(--vf-node-color, #0041d0);
}

.vue-flow__node-default {
  --vf-handle: var(--vf-node-color, #1a192b);
  --vf-box-shadow: var(--vf-node-color, #1a192b);

  background: var(--vf-node-bg);
  border-color: var(--vf-node-color, #1a192b);
}

.vue-flow__node-default.selected,
.vue-flow__node-default:focus,
.vue-flow__node-default:focus-visible {
    outline: none;
    border: 1px solid var(--vf-node-color, #1a192b);
}

.vue-flow__node-output {
  --vf-handle: var(--vf-node-color, #ff0072);
  --vf-box-shadow: var(--vf-node-color, #ff0072);

  background: var(--vf-node-bg);
  border-color: var(--vf-node-color, #ff0072);
}

.vue-flow__node-output.selected,
.vue-flow__node-output:focus,
.vue-flow__node-output:focus-visible {
    outline: none;
    border: 1px solid var(--vf-node-color, #ff0072);
}

.vue-flow__nodesselection-rect,
.vue-flow__selection {
  background: rgba(0, 89, 220, 0.08);
  border: 1px dotted rgba(0, 89, 220, 0.8);
}

.vue-flow__nodesselection-rect:focus,
.vue-flow__nodesselection-rect:focus-visible,
.vue-flow__selection:focus,
.vue-flow__selection:focus-visible {
  outline: none;
}

.vue-flow__handle {
  width: 18px;
  height: 8px;
  background: #000;
  border: 1px solid #fff;
  border-radius: 3px;
}

.vue-flow__controls{box-shadow:0 0 2px 1px #00000014}.vue-flow__controls-button{background:#fefefe;border:none;border-bottom:1px solid #eee;box-sizing:content-box;display:flex;justify-content:center;align-items:center;width:16px;height:16px;cursor:pointer;user-select:none;padding:5px}.vue-flow__controls-button svg{max-width:12px;max-height:12px;overflow:visible}.vue-flow__controls-button:hover{background:#f4f4f4}.vue-flow__controls-button:disabled{pointer-events:none}.vue-flow__controls-button:disabled svg{fill-opacity:.4}


// const flags = ['', 'red-flag', 'blue-flag', 'green-flag', 
//'yellow-flag', 'orange-flag', 'purple-flag', 'pink-flag', 
//'brown-flag', 'black-flag', 'white-flag'];


.red-flag{border:2px solid red !important;}
.blue-flag{border:2px solid blue !important;}
.green-flag{border:2px solid green !important;}
.yellow-flag{border:2px solid yellow !important;}
.orange-flag{border:2px solid orange !important;}
.purple-flag{border:2px solid purple !important;}
.pink-flag{border:2px solid pink !important;}
.brown-flag{border:2px solid brown !important;}
.black-flag{border:2px solid black !important;}
.white-flag{border:2px solid white !important;}




input[type=range] {
  -webkit-appearance: none; /* Hides the slider so that custom slider can be made */
  width: 82%; /* Specific width is required for Firefox. */
  background: transparent; /* Otherwise white in Chrome */
  margin-top: 15px;
}

input[type=range]::-webkit-slider-thumb {
  -webkit-appearance: none;
}

input[type=range]::-webkit-slider-thumb {
  -webkit-appearance: none;
  border: 1px solid #555;
  height: 20px;
  width: 20px;
  border-radius: 20%;
  background: var(--vf-node-bg);
  cursor: pointer;
  margin-top: -9px; /* You need to specify a margin in Chrome, but in Firefox and IE it is automatic */
}

/* All the same stuff for Firefox */
input[type=range]::-moz-range-thumb {
  border: 1px solid #555;
  height: 20px;
  width: 20px;
  border-radius: 20%;
  background: var(--vf-node-bg);
  cursor: pointer;
}

/* All the same stuff for IE */
input[type=range]::-ms-thumb {
  border: 1px solid #555;
  height: 20px;
  width: 20px;
  border-radius: 20%;
  background: var(--vf-node-bg);
  cursor: pointer;
}

input[type=range]::-webkit-slider-runnable-track {
  width: 82%;
  height: 4px;
  cursor: pointer;
  background: #b1b1b7;
  border-radius: 1px;
  border: 0.2px solid #010101;
}

input[type=range]::-moz-range-track {
  width: 82%;
  height: 4px;
  cursor: pointer;
  background: #b1b1b7;
  border-radius: 1px;
  border: 0.2px solid #010101;
}

input[type=range]::-ms-track {
  width: 82%;
  height: 4px;
  cursor: pointer;
  background: transparent;
  border-color: transparent;
  color: transparent;
}

input[type=range]::-ms-fill-lower {
  background: #b1b1b7;
  border: 0.2px solid #010101;
  border-radius: 10px;
}

input[type=range]::-ms-fill-upper {
  background: #b1b1b7;
  border: 0.2px solid #010101;
  border-radius: 10px;
}
</style>
