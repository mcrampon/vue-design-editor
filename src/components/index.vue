<template>
  <div class="wrapper" @mouseup="onMouseUp">
    <tools :stories="stories" :active-tool="activeTool" @select-tool="selectTool" @mounted="updateToolsAfterMount" />

    <div
      ref="canvasWrapper"
      class="canvas-wrapper"
      @wheel="onWheel"
      @mouseenter="onEnter"
      @focus="onEnter"
      @mousemove="onMove"
      @mouseleave="onLeave"
      @blur="onLeave"
      @click="onClick"
      @mousedown="onMouseDown"
    >
      <div ref="canvas" class="canvas">
        <div ref="floatingCompRef" class="floating-component-wrapper">
          <div v-if="floatingComp" class="floating-comp">
            <component :is="floatingComp" />
          </div>
        </div>
        <embedded-component
          v-for="(embeddedComponent, index) in embeddedComponents"
          :key="index"
          :embedded-component="embeddedComponent"
          :selected="index === selectedEmbeddedIndex"
          :interactable="activeTool === 'mouse'"
          :canvas-scale="canvasScale"
          :slot-in-edit-mode="slotInEditMode"
          @mounted="
            instance => updateEmbeddedComponentAfterMount(index, instance)
          "
          @resize="onResize"
          @mousedown="e => onMouseDownEmbedded(index, e)"
        />
      </div>
    </div>

    <component-config
      v-if="selectedComponent"
      :component="selectedComponent"
      @update-prop="updateProp"
      @toggle-slot="toggleSlot"
      @edit-slot="editSlot"
    />
  </div>
</template>

<script>
import { markRaw } from 'vue';

import EmbeddedComponent from './embedded_component.vue';
import ComponentConfig from './component_config.vue';
import Tools from './tools.vue';

export default {
  components: {
    EmbeddedComponent,
    ComponentConfig,
    Tools
  },

  props: {
    stories: {
      type: Array,
      required: true
    }
  },

  data: function () {
    return {
      tools: [],
      activeTool: '',
      floatingComp: null,
      rafId: null,
      embeddedComponents: [],
      selectedEmbeddedIndex: -1,
      moving: false,
      moveOffset: { x: 0, y: 0 },
      resizingCorner: '',
      slotInEditMode: '',
      canvasScale: 1,
      canvasOffset: { x: 0, y: 0 },
      spacePressed: false,
      panning: false,
      panStart: { x: 0, y: 0 },
      panStartOffset: { x: 0, y: 0 },
      clickEnabled: true
    };
  },

  computed: {
    selectedComponent: function () {
      return this.embeddedComponents[this.selectedEmbeddedIndex];
    }
  },

  mounted() {
    document.addEventListener('keydown', this.onKeyDown);
    document.addEventListener('keyup', this.onKeyUp);
  },

  beforeUnmount() {
    document.removeEventListener('keydown', this.onKeyDown);
    document.removeEventListener('keyup', this.onKeyUp);
  },

  methods: {
    wrapperOffset: function () {
      const rect = this.$refs.canvasWrapper.getBoundingClientRect();
      return {
        x: rect.x,
        y: rect.y
      };
    },

    selectTool: function (tool) {
      if (this.activeTool === 'mover' && tool !== 'mover') {
        this.selectedEmbeddedIndex = -1;
      }

      this.activeTool = tool;
    },

    onEnter: function () {
      const story = this.tools[this.activeTool];

      if (!story?.render) {
        this.floatingComp = null;
      } else {
        this.floatingComp = markRaw(
          story.render(this.initialStoryProps(story))
        );
      }
    },

    onMove: function (e) {
      if (this.panning) {
        return this.panCanvas(e);
      }
      if (this.resizingCorner) {
        if (this.slotInEditMode) {
          return this.resizeSlot(e);
        }
        return this.resizeSelected(e);
      }
      if (this.moving && !this.slotInEditMode) {
        return this.moveSelected(e);
      }
      if (this.floatingComp) {
        return this.moveFloatingComp(e);
      }
    },

    resizeSelected: function (e) {
      if (!this.selectedComponent) return;

      const el = this.selectedComponent.el;
      const rect = el.getBoundingClientRect();
      const wrapperOffset = this.wrapperOffset();

      let { width, height, transform } = this.selectedComponent.canvasData;
      let x, y;

      // Convert screen coordinates to canvas coordinates
      const mouseCanvasX =
        (e.clientX - wrapperOffset.x - this.canvasOffset.x) / this.canvasScale;
      const mouseCanvasY =
        (e.clientY - wrapperOffset.y - this.canvasOffset.y) / this.canvasScale;
      const rectCanvasX =
        (rect.x - wrapperOffset.x - this.canvasOffset.x) / this.canvasScale;
      const rectCanvasY =
        (rect.y - wrapperOffset.y - this.canvasOffset.y) / this.canvasScale;
      const rectCanvasWidth = rect.width / this.canvasScale;
      const rectCanvasHeight = rect.height / this.canvasScale;

      switch (this.resizingCorner) {
        case 'top-left':
          width = rectCanvasWidth + rectCanvasX - mouseCanvasX;
          height = rectCanvasHeight + rectCanvasY - mouseCanvasY;
          x = mouseCanvasX + 'px';
          y = mouseCanvasY + 'px';
          transform = `translate(${x}, ${y})`;
          break;
        case 'top-center':
          height = rectCanvasHeight + rectCanvasY - mouseCanvasY;
          x = rectCanvasX + 'px';
          y = mouseCanvasY + 'px';
          transform = `translate(${x}, ${y})`;
          break;
        case 'top-right':
          width = mouseCanvasX - rectCanvasX;
          height = rectCanvasHeight + rectCanvasY - mouseCanvasY;
          x = rectCanvasX + 'px';
          y = mouseCanvasY + 'px';
          transform = `translate(${x}, ${y})`;
          break;
        case 'center-right':
          width = mouseCanvasX - rectCanvasX;
          break;
        case 'bottom-right':
          width = mouseCanvasX - rectCanvasX;
          height = mouseCanvasY - rectCanvasY;
          break;
        case 'bottom-center':
          height = mouseCanvasY - rectCanvasY;
          break;
        case 'bottom-left':
          width = rectCanvasWidth + rectCanvasX - mouseCanvasX;
          height = mouseCanvasY - rectCanvasY;
          x = mouseCanvasX + 'px';
          y = rectCanvasY + 'px';
          transform = `translate(${x}, ${y})`;
          break;
        case 'center-left':
          width = rectCanvasWidth + rectCanvasX - mouseCanvasX;
          x = mouseCanvasX + 'px';
          y = rectCanvasY + 'px';
          transform = `translate(${x}, ${y})`;
          break;
      }

      this.selectedComponent.canvasData = {
        ...this.selectedComponent.canvasData,
        width,
        height,
        transform
      };
    },

    resizeSlot: function (e) {
      // TODO: could probably DRY things up quite a lot with `resizeSelected`
      if (!this.slotInEditMode) return;

      const slot = this.selectedComponent.canvasData.slots[this.slotInEditMode];
      const el =
        this.selectedComponent.instance.$refs[`slot:${this.slotInEditMode}`];
      const rect = el.getBoundingClientRect();
      const wrapperOffset = this.wrapperOffset();

      let { width, height } = slot.canvasData;

      // Convert screen coordinates to canvas coordinates
      const mouseCanvasX =
        (e.clientX - wrapperOffset.x - this.canvasOffset.x) / this.canvasScale;
      const mouseCanvasY =
        (e.clientY - wrapperOffset.y - this.canvasOffset.y) / this.canvasScale;
      const rectCanvasX =
        (rect.x - wrapperOffset.x - this.canvasOffset.x) / this.canvasScale;
      const rectCanvasY =
        (rect.y - wrapperOffset.y - this.canvasOffset.y) / this.canvasScale;

      switch (this.resizingCorner) {
        case 'center-right':
          width = mouseCanvasX - rectCanvasX;
          break;
        case 'bottom-right':
          width = mouseCanvasX - rectCanvasX;
          height = mouseCanvasY - rectCanvasY;
          break;
        case 'bottom-center':
          height = mouseCanvasY - rectCanvasY;
          break;
      }

      slot.canvasData = {
        ...slot.canvasData,
        width,
        height
      };
    },

    moveSelected: function (e) {
      if (!this.selectedComponent) return;
      this.selectedComponent.canvasData.transform = this.componentTransform(e);
    },

    moveFloatingComp: function (e) {
      const el = this.$refs.floatingCompRef;
      if (!el) return;

      cancelAnimationFrame(this.rafId);
      this.rafId = requestAnimationFrame(() => {
        el.style.transform = this.componentTransform(e);
      });
    },

    onClick: function (e) {
      if (this.panning) return;
      if (!this.clickEnabled) {
        this.clickEnabled = true;
        return;
      }
      if (
        this.activeTool === 'mover' &&
        e.target === this.$refs.canvasWrapper
      ) {
        this.selectedEmbeddedIndex = -1;
        this.slotInEditMode = '';
      }
      if (!this.floatingComp) return;

      const story = this.tools[this.activeTool];

      this.embeddedComponents.push({
        render: story.render,
        props: this.initialStoryProps(story),
        canvasData: {
          definition: story.definition,
          argTypes: story.argTypes,
          availableSlots: story.availableSlots,
          transform: this.componentTransform(e),
          width: 'auto',
          height: 'auto'
        }
      });
    },

    onMouseDown: function (e) {
      if (this.spacePressed) {
        this.startPan(e);
        this.clickEnabled = false;
      }
    },

    componentTransform: function (e) {
      const wrapperOffset = this.wrapperOffset();
      // Convert screen coordinates to canvas coordinates
      const x =
        (e.clientX - wrapperOffset.x - this.canvasOffset.x) / this.canvasScale;
      const y =
        (e.clientY - wrapperOffset.y - this.canvasOffset.y) / this.canvasScale;

      const anchor = {};

      if (this.moving) {
        anchor.x = -(this.moveOffset.x + 2) + 'px';
        anchor.y = -(this.moveOffset.y + 2) + 'px';
      } else {
        anchor.x = '-50%';
        anchor.y = '-50%';
      }

      return `translate(${anchor.x}, ${anchor.y}) translate(${x}px, ${y}px)`;
    },

    onLeave: function () {
      this.floatingComp = null;
      this.moving = false;
      this.resizingCorner = '';
      this.panning = false;
    },

    onMouseDownEmbedded: function (index, e) {
      if (this.spacePressed) {
        this.startPan(e);
        return;
      }

      if (this.activeTool !== 'mover') {
        return;
      }

      this.selectedEmbeddedIndex = index;
      this.moving = true;
      this.moveOffset = { x: e.offsetX, y: e.offsetY };
    },

    onMouseUp: function () {
      if (this.panning) {
        this.stopPan();
        return;
      }

      if (this.activeTool === 'mover') {
        this.moving = false;
        this.resizingCorner = '';
      }
    },

    onResize: function (e, corner) {
      this.resizingCorner = corner;
      this.clickEnabled = false;
    },

    updateEmbeddedComponentAfterMount: function (index, instance) {
      const component = this.embeddedComponents[index];
      const el = instance.$refs.wrapper;

      // Update those to have easy access to everything
      component.instance = instance;
      component.el = el;

      // Get rid of -50%/-50% transform: it's annoying when the component is set in the canvas.
      const rect = el.getBoundingClientRect();
      const wrapperOffset = this.wrapperOffset();

      // Convert screen coordinates to canvas coordinates
      const x =
        (rect.x - wrapperOffset.x - this.canvasOffset.x) / this.canvasScale +
        'px';
      const y =
        (rect.y - wrapperOffset.y - this.canvasOffset.y) / this.canvasScale +
        'px';

      this.embeddedComponents[index].canvasData.transform =
        `translate(${x}, ${y})`;
    },

    updateToolsAfterMount: function (tools) {
      this.tools = tools;
    },

    updateProp: function (argName, value) {
      this.selectedComponent.props[argName] = value;
    },

    toggleSlot: function (slotName) {
      if (this.selectedComponent.canvasData.slots?.[slotName]) {
        delete this.selectedComponent.canvasData.slots[slotName];
      } else {
        this.selectedComponent.canvasData.slots ||= {};
        this.selectedComponent.canvasData.slots[slotName] = {
          canvasData: {
            width: 'auto',
            height: 'auto'
          },
          render: () => ({
            template: `<div data-slot-name="${slotName}"></div>`
          })
        };
      }
    },

    editSlot: function (slotName) {
      // TODO: make the slot the new canvas and insert new components inside
      this.slotInEditMode = slotName;
    },

    onWheel: function (e) {
      e.preventDefault();

      const wrapperRect = this.$refs.canvasWrapper.getBoundingClientRect();

      // Get mouse position relative to canvas wrapper
      const mouseX = e.clientX - wrapperRect.left;
      const mouseY = e.clientY - wrapperRect.top;

      // Convert mouse position to canvas coordinates (before zoom)
      const canvasMouseX = (mouseX - this.canvasOffset.x) / this.canvasScale;
      const canvasMouseY = (mouseY - this.canvasOffset.y) / this.canvasScale;

      const zoomSpeed = 1.1;
      this.canvasScale *= e.deltaY < 0 ? zoomSpeed : 1 / zoomSpeed;
      this.canvasScale = Math.min(Math.max(0.1, this.canvasScale), 20);

      // Calculate new canvas position to keep the point under cursor stationary
      this.canvasOffset.x = mouseX - canvasMouseX * this.canvasScale;
      this.canvasOffset.y = mouseY - canvasMouseY * this.canvasScale;

      this.$refs.canvas.style.transform = `translate(${this.canvasOffset.x}px, ${this.canvasOffset.y}px) scale(${this.canvasScale})`;
    },

    onKeyDown: function (e) {
      const formElements = ['INPUT', 'TEXTAREA', 'SELECT', 'OPTION'];
      if (formElements.includes(e.target.tagName)) {
        // Allow typing
        return;
      }

      if (e.code === 'Space') {
        e.preventDefault();
        this.spacePressed = true;
        this.$refs.canvasWrapper.style.cursor = 'grab';
      } else if (e.code === 'Delete' && this.selectedComponent) {
        this.embeddedComponents.splice(this.selectedEmbeddedIndex, 1);
        this.selectedEmbeddedIndex = -1;
      }
    },

    onKeyUp: function (e) {
      const formElements = ['INPUT', 'TEXTAREA', 'SELECT', 'OPTION'];
      if (formElements.includes(e.target.tagName)) {
        // Allow typing
        return;
      }

      if (e.code === 'Space') {
        e.preventDefault();
        this.spacePressed = false;
        this.panning = false;
        this.$refs.canvasWrapper.style.cursor = '';
      }
    },

    startPan: function (e) {
      this.panning = true;
      this.panStart = { x: e.clientX, y: e.clientY };
      this.panStartOffset = { ...this.canvasOffset };
      this.$refs.canvasWrapper.style.cursor = 'grabbing';
    },

    stopPan: function () {
      this.panning = false;
      this.$refs.canvasWrapper.style.cursor = 'grab';
    },

    panCanvas: function (e) {
      if (!this.panning) return;

      const deltaX = e.clientX - this.panStart.x;
      const deltaY = e.clientY - this.panStart.y;

      this.canvasOffset.x = this.panStartOffset.x + deltaX;
      this.canvasOffset.y = this.panStartOffset.y + deltaY;

      this.$refs.canvas.style.transform = `translate(${this.canvasOffset.x}px, ${this.canvasOffset.y}px) scale(${this.canvasScale})`;
    },

    initialStoryProps: function (story) {
      return { ...(story.definition.args || {}), ...(story.args || {}) };
    }
  }
};
</script>

<style scoped>
.wrapper {
  display: flex;
  flex-direction: row;
  height: 100vh;
}

.canvas-wrapper {
  flex-grow: 1;
  width: 100%;
  height: 100%;
  overflow: hidden;
  position: relative;
  background-color: #232323;
}

.canvas {
  position: absolute;
  top: 0;
  left: 0;
  transform-origin: 0 0;

  .floating-component-wrapper {
    pointer-events: none;
    position: absolute;
    opacity: 0.5;

    .floating-comp {
      pointer-events: none;
    }
  }
}
</style>
