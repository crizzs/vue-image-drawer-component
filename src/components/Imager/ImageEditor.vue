<template>
  <div class="image-editor">
    <div class="container-fluid">
      <div class="edit-shape d-flex text-center my-3 justify-content-center flex-wrap text-capitalize">
        <template v-if="controls.shape !== 'text'">
        <div class="d-flex flex-column align-items-center border mb-2 p-2 card">
            <label for="line-width">Line width</label>
            <input type="range" id="line-width" :min="1" v-model="controls.lineWidth" :max="50" />
          </div>
        <div class="d-flex flex-column align-items-center border mb-2 p-2 card">
            <label for="stroke-color">stroke color</label>
            <input type="color" id="stroke-color" v-model="controls.strokeColor"/>
          </div>
        </template>
        <template v-else>
        <div class="d-flex flex-column align-items-center border mb-2 p-2 card">
            <label for="font-size">Font size</label>
            <input type="range" id="font-size" :min="1" v-model="controls.fontSize" :max="50" />
          </div>
        <div class="d-flex flex-column align-items-center border mb-2 p-2 card">
            <label for="text-color">text color</label>
            <input type="color" id="text-color" v-model="controls.textColor"/>
          </div>
        </template>
        <div class="d-flex flex-column align-items-center border mb-2 p-2 card">
          <label for="text-color">undo</label>
          <button 
            class="btn btn-primary btn-sm"
            :class="{disabled: !undoBtnActivation}"
            :disabled="!undoBtnActivation"
            @click="undoShapes">
              <i class="fa fa-undo"></i>
          </button>
        </div>
        <div class="d-flex flex-column align-items-center border mb-2 p-2 card">
          <label for="text-color">redo</label>
          <button
            class="btn btn-primary btn-sm"
            :class="{disabled: !shapesStack.redoStackDataStore.length}"
            :disabled="!shapesStack.redoStackDataStore.length"
            @click="redoShapes">
              <i class="fa fa-undo" style="transform: rotateY(180deg)"></i>
            </button>
        </div>
        <div class="d-flex flex-column align-items-center border mb-2 p-2 card">
          <label for="text-color">Close</label>
          <button class="btn btn-primary btn-sm" @click="$emit('input', null)">
            <i class="fa fa-close"></i>
          </button>
        </div>
        <div class="d-flex flex-column align-items-center border mb-2 p-2 card">
          <label for="text-color">Reset Image</label>
        <button class="btn btn-primary btn-sm" @click="clearImage">
          <i class="fa fa-trash"></i>
        </button>
        </div>
        <div class="d-flex flex-column align-items-center border mb-2 p-2 card">
          <label for="text-color">Save Image</label>
        <button class="btn btn-primary btn-sm" @click="saveCanvasAsImage">
          <i class="fa fa-save"></i>
        </button>
        </div>
      </div>
      <div class="image-editor-wrapper">
        <div class="row">
          <div class="col-lg-1 col-md-2">
            <div class="image-editor-toolbar bar-bg d-flex flex-md-column align-items-md-center align-items-center justify-content-around">
              <EditorShape v-model="controls.shape" shape="circle" icon="genderless"/>
              <EditorShape v-model="controls.shape" shape="line" icon="ellipsis-v"/>
              <EditorShape v-model="controls.shape" shape="pen" icon="pencil"/>
              <EditorShape v-model="controls.shape" shape="text" icon="font"/>
              <EditorShape v-model="controls.shape" shape="arrow" icon="long-arrow-down"/>
            </div>
          </div>
            <div class="col-lg-9 col-md-10">
              <div class="image-editor-canvas">
                <canvas
                  ref="canvas"
                  width="800"
                  height="800"
                  @mousemove="onDragging"
                  @mouseup="onDragStop"
                  @mousedown="onDragStart"

                  @touchmove.passive="onDragging"
                  @touchend="onDragStop"
                  @touchstart.passive="onDragStart"
                  >
                </canvas>
                <div ref="canvas_text" class="canvas-text">
                  <textarea
                    @blur="saveCanvasText"
                    width="0"
                    v-model="canvas_text"
                    :style="{
                      color: controls.textColor,
                      fontSize: controls.fontSize+'px',
                      borderColor: controls.strokeColor,
                      backgroundColor: controls.fillBox ? controls.fillColor : 'transparent' }">
                  </textarea>
                </div>
              </div>
            </div>
          <div class="col-lg-2" v-if="shapesStack.dataStore.length">
            <div id="accordion">
              <LayerText
                :value="listShapesALayer('text')"
                v-if="listShapesALayer('text').length"
                @onDelete="deleteShape"
                @textUpdated="textUpdated"
                 />
              <LayerLine
                :value="listShapesALayer('line')"
                v-if="listShapesALayer('line').length"
                @onDelete="deleteShape"
              />
              <LayerCircle
                :value="listShapesALayer('circle')"
                v-if="listShapesALayer('circle').length"
                @onDelete="deleteShape"
              />
              <LayerArrow
                :value="listShapesALayer('arrow')"
                v-if="listShapesALayer('arrow').length"
                @onDelete="deleteShape"
              />
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
import EditorShape from "./EditorShape";
// Shapes
import drawCircle from "./shapes/circle";
import drawArrow from "./shapes/arrow";
import drawLine from "./shapes/line";
import { drawTextarea, saveCanvasText, startTextFrom, redrawText } from "./shapes/text";
import pen from "./shapes/pen";
// Draw Events
import onDragStart from "./events/onDragStart";
import onDragStop from "./events/onDragStop";
import onDragging from "./events/onDragging";
// Layers
import LayerText from './layers/LayerText';
import LayerLine from './layers/LayerLine';
import LayerCircle from './layers/LayerCircle';
import LayerArrow from './layers/LayerArrow';

export default {
  components: { EditorShape, LayerText, LayerLine, LayerCircle, LayerArrow },
  props: {
    value: {}
  },
  data() {
    return {
      canvas: null,
      ctx: null,
      dragging: false,
      snapshot: null,
      dragStartLocation: null,
      dragStartLocationEvent: null,
      dragStartLocationTemp: null,
      canvas_text: null,
      controls: {
        fillBox: false,
        shape: "text",
        lineWidth: 6,
        strokeColor: "#800080",
        fillColor: "#800080",
        textColor: "#800080",
        fontSize: 40
      },
      shapesStack: {
        dataStore: [],
        redoStackDataStore: [],
        holderDataStore: []
      },
      isOnMobile: false
    };
  },
  computed: {
    imageSizes() {
      const img = new Image();
      img.src = this.value.src;
      return {
        widt: img.width,
        hegh: img.height
      };
    },
    undoBtnActivation () {
      if (this.shapesStack.dataStore.length) {
        return true
      } else if (this.shapesStack.holderDataStore.length) {
        return true
      }
      return false
    }
  },
  mounted() {
    (function(vm) {
      vm.init();
      window.addEventListener("keyup", vm.ctrlZ);
    })(this);
  },
  methods: {
    ctrlZ(event) {
      if (event.keyCode == 90 && event.ctrlKey && event.keyCode === 16) {
        this.redoShapes()
      } else if (event.keyCode == 90 && event.ctrlKey) {
        this.undoShapes()
      }
    },
    reRenderShapes(type = 'undo') {
      this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
      const self = this;
      let shapes = self.shapesStack.dataStore;
      this.initCanvasBg(() => {
        for (let i = 0; i < shapes.length; i++) {
          let shape = shapes[i]

          switch (shape.type) {
            case 'text':
              redrawText(shape, self)
              break;
            case 'line':
              drawLine(shape, self.ctx);
              break;
            case 'circle':
              drawCircle(shape, self.ctx)
              break;
            case 'arrow':
              drawArrow(shape, self.ctx)
              break;
          }
        }
      });
    },
    init() {
      this.canvas = this.$refs.canvas;
      // make canvas takes its parent width
      // init canvas context
      this.ctx = this.canvas.getContext("2d");
      const c_div = document.querySelector(".image-editor-canvas");
      // shapes style
      this.ctx.strokeStyle = this.controls.strokeColor;
      this.ctx.fillStyle = this.controls.fillColor;
      this.ctx.lineWidth = this.controls.lineWidth;
      this.ctx.ClipToBounds = true;
      this.ctx.lineCap = "round";
      this.initCanvasBg();
    },
    listShapesALayer(shape){
      const arr = this.shapesStack.dataStore.filter(el => el.type === shape);
      return arr;
    },
    deleteShape(key) {
      const shapeIndex = this.shapesStack.dataStore.findIndex(shape => shape.timestamp === key);
      this.shapesStack.holderDataStore.push(this.shapesStack.dataStore[shapeIndex]);
      this.$set(this.shapesStack.dataStore[shapeIndex], "isDeleted", true);
      this.shapesStack.dataStore.splice(shapeIndex, 1);
      this.reRenderShapes()
    },
    textUpdated (updateTextProps) {
     const elIndex = this.shapesStack.dataStore.findIndex(el => el.timestamp === updateTextProps.timestamp);
     this.shapesStack.holderDataStore.push(this.shapesStack.dataStore[elIndex]);
     this.shapesStack.dataStore.splice(elIndex, 1);
     this.push(updateTextProps);
     this.reRenderShapes();
    },
    initCanvasBg(callback) {
      // draw image on canvas
      const img = new Image();
      img.src = this.value.src;
      img.onload = () => {
        this.canvas.width = img.width
        this.canvas.height = img.height
        this.canvas.height = this.canvas.offsetHeight
        this.canvas.width = this.canvas.offsetWidth

        this.ctx.drawImage(img, 0, 0, this.canvas.offsetWidth, this.canvas.offsetHeight);
        if (callback) {
          callback(true);
        }
      };
    },
    takeSnapshot() {
      this.snapshot = this.ctx.getImageData(
        0,
        0,
        this.canvas.width,
        this.canvas.height
      );
    },
    getSnapshot() {
      this.ctx.putImageData(this.snapshot, 0, 0);
    },
    clearImage() {
      this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
      this.initCanvasBg();
      this.shapesStack.holderDataStore = [...this.shapesStack.dataStore]
      this.shapesStack.dataStore = []
    },
    getCanvasCoordinatesMouse(mouseEvent) {
      const x = mouseEvent.clientX - this.canvas.getBoundingClientRect().left;
      const y = mouseEvent.clientY - this.canvas.getBoundingClientRect().top;
      return {
        x,
        y
      };
    }
    ,
    getCanvasCoordinates(event) {
      if (this.isOnMobile) {
        return this.getCanvasCoordinatesTouch(event);
      }
      return this.getCanvasCoordinatesMouse(event);
    },
    getCanvasCoordinatesTouch(touchEvent) {
      const clientX = (touchEvent.targetTouches[0] ? touchEvent.targetTouches[0].pageX : touchEvent.changedTouches[touchEvent.changedTouches.length-1].pageX)
      const clientY = (touchEvent.targetTouches[0] ? touchEvent.targetTouches[0].pageY : touchEvent.changedTouches[touchEvent.changedTouches.length-1].pageY)
      const x = clientX - this.canvas.getBoundingClientRect().left;
      const y = clientY - this.canvas.getBoundingClientRect().top;
      touchEvent.preventDefault();
      return {
        x,
        y
      };
    },
    // Mouse Events
    onDragStart(event) {
      if (this.controls.shape === 'text') {
        startTextFrom(this);
      }
      return onDragStart(this)(event);
    },
    onDragging(event) {
      return onDragging(this)(event);
    },
    onDragStop(event) {
      return onDragStop(this, pen)(event);
    },

    draw(event) {
        const pos = this.getCanvasCoordinates(event);
      this.getSnapshot();
      switch (this.controls.shape) {
        case "line":
          drawLine(
            {
              type: this.controls.shape,
              from: this.dragStartLocation,
              to: pos,
              strokeColor: this.controls.strokeColor,
              lineWidth: this.controls.lineWidth
            },
            this.ctx
          );
          break;
        case "circle":
          drawCircle({
            from: this.dragStartLocation,
            to: pos,
            strokeColor: this.controls.strokeColor,
            lineWidth: this.controls.lineWidth
            }, this.ctx);
          break;
        case "pen":
          pen.drawPen({
            to: pos,
            from: this.dragStartLocation,
            strokeColor: this.controls.strokeColor,
            lineWidth: this.controls.lineWidth
            }, this.ctx);
          break;
        case "text":
        this.$nextTick(() => {
          drawTextarea(
            { from: this.dragStartLocation, to: pos },
            this
          );
        })
          break;
        case "arrow":
          drawArrow({ from: this.dragStartLocation, to: pos }, this.ctx);
          break;
      }
    },

    checkIsTouchEvent(event) {
      if (event.type.includes('touch')) {
        this.isOnMobile = true;
        return true;
      }
      this.isOnMobile = false;
      return false;
    },

    saveCanvasAsImage() {
      const imgDataUrl = this.canvas.toDataURL("image/png");
      this.$nextTick(() => {
        this.$emit("input", { src: imgDataUrl, key: this.value.key });
      });
    },
    saveCanvasText(event) {
      return saveCanvasText({ from: this.dragStartLocationTemp }, this.ctx, this);
    },

    push (element) {
      this.shapesStack.dataStore.push(element);
    },
    pop () {
      const pop = this.shapesStack.dataStore.pop();
      return pop;
    },
    redoPush (element) {
      this.shapesStack.redoStackDataStore.push(element);
    },
    redoPop () {
      const pop = this.shapesStack.redoStackDataStore.pop()
      return pop;
    },
    undoShapes() {
      if (this.shapesStack.holderDataStore.length) {
        const shape = this.shapesStack.holderDataStore.pop();
        if (shape.type === "text" && !shape.isDeleted) {
          const undoShape = this.pop();
          this.redoPush(undoShape);
        }
        this.push(shape)
        this.reRenderShapes()
      }
      else if (this.shapesStack.dataStore.length) {
        const shape = this.pop();
        if (!shape) return;
        this.redoPush(shape);
        this.reRenderShapes()
      }
    },
    redoShapes() {
      if (this.shapesStack.redoStackDataStore.length) {
        const shape = this.redoPop();
        this.push(shape);
        this.reRenderShapes('redo')
      }
    }
  },
  watch: {
    "controls.lineWidth": function(nVal, oldVal) {
      this.ctx.lineWidth = nVal;
    },
    "controls.strokeColor": function(nVal, oldVal) {
      this.ctx.strokeStyle = nVal;
      this.ctx.fillStyle = nVal;
    },
    "controls.shape": function(nVal, oldVal) {
      this.$refs.canvas_text.style.display = "none";
    }
  }
};
</script>
