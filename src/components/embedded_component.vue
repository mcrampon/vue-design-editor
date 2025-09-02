<template>
  <div
    ref="wrapper"
    class="selectable-component-wrapper"
    :class="{ selected }"
    :style="{
      transform,
      width,
      height,
      '--borderSize': borderSize,
      '--handleSize': handleSize,
      '--offsetSize': offsetSize
    }"
  >
    <div v-if="selected && !slotInEditMode" class="resizers-wrapper">
      <div
        v-for="corner in resizers"
        :key="corner"
        :class="corner"
        draggable="false"
        @mousedown="e => onMouseDown(e, corner)"
      />
    </div>
    <div class="embedded-comp" :class="{ interactable }">
      <component :is="rendered">
        <!-- Treat default slot separately because it can't be named -->
        <template v-if="defaultSlot">
          <div
            ref="slot:default"
            class="slot"
            :style="{
              width: slotWidth('default'),
              height: slotHeight('default')
            }"
          >
            <div v-if="editingSlot('default')" class="resizers-wrapper">
              <div
                v-for="corner in slotResizers"
                :key="corner"
                :class="corner"
                draggable="false"
                @mousedown="e => onMouseDown(e, corner)"
              />
            </div>
            <component :is="renderedSlot(defaultSlot)" />
          </div>
        </template>
        <template v-for="slot in nonDefaultSlots" :key="slot" #[slot]>
          <div
            class="slot"
            :style="{ width: slotWidth(slot), height: slotHeight(slot) }"
          >
            <div v-if="editingSlot(slot)" class="resizers-wrapper">
              <div
                v-for="corner in slotResizers"
                :key="corner"
                :class="corner"
                draggable="false"
                @mousedown="e => onMouseDown(e, corner)"
              />
            </div>
            <component :is="renderedSlot(slot)" />
          </div>
        </template>
      </component>
    </div>
  </div>
</template>

<script>
const resizers = [
  'top-left',
  'top-center',
  'top-right',
  'center-right',
  'bottom-right',
  'bottom-center',
  'bottom-left',
  'center-left'
];

const slotResizers = ['center-right', 'bottom-right', 'bottom-center'];

export default {
  props: {
    embeddedComponent: {
      type: Object,
      required: true
    },

    selected: {
      type: Boolean,
      required: true
    },

    interactable: {
      type: Boolean,
      required: true
    },

    canvasScale: {
      type: Number,
      default: 1
    },

    slotInEditMode: {
      type: String,
      default: ''
    }
  },

  emits: ['mounted', 'resize'],

  data: function () {
    return {
      resizers,
      slotResizers
    };
  },

  computed: {
    rendered: function () {
      return this.embeddedComponent.render(this.embeddedComponent.props);
    },

    nonDefaultSlots: function () {
      const defaultSlotName = 'default';
      // eslint-disable-next-line no-unused-vars
      const { [defaultSlotName]: _, ...rest } =
        this.embeddedComponent.canvasData.slots || {};
      return rest;
    },

    defaultSlot: function () {
      return this.embeddedComponent.canvasData.slots?.default;
    },

    transform: function () {
      return this.embeddedComponent.canvasData.transform;
    },

    width: function () {
      return this.size(this.embeddedComponent.canvasData.width);
    },

    height: function () {
      return this.size(this.embeddedComponent.canvasData.height);
    },

    borderSize: function () {
      return Math.max(1, 2 / this.canvasScale) + 'px';
    },

    handleSize: function () {
      return Math.max(4, 16 / this.canvasScale) + 'px';
    },

    offsetSize: function () {
      return -Math.max(2, 8 / this.canvasScale) + 'px';
    }
  },

  mounted: function () {
    this.$emit('mounted', this);
  },

  methods: {
    onMouseDown: function (e, corner) {
      e.stopPropagation();
      this.$emit('resize', e, corner);
    },

    renderedSlot: function (slot) {
      return slot.render();
    },

    editingSlot: function (slot) {
      return this.slotInEditMode === slot;
    },

    slotWidth: function (slot) {
      return this.size(
        this.embeddedComponent.canvasData.slots[slot]?.canvasData?.width
      );
    },

    slotHeight: function (slot) {
      return this.size(
        this.embeddedComponent.canvasData.slots[slot]?.canvasData?.height
      );
    },

    size: function (s) {
      if (s === 'auto') return 'auto';
      return `${s}px`;
    }
  }
};
</script>

<style scoped>
.selectable-component-wrapper {
  position: absolute;
  min-width: 4px;
  min-height: 4px;

  &.selected {
    .resizers-wrapper {
      pointer-events: initial;
      z-index: 10;
      border: var(--borderSize) solid #0a89ff;
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      user-drag: none;
      -webkit-user-drag: none;
      -moz-user-select: none;

      & > * {
        position: absolute;
        width: var(--handleSize);
        height: var(--handleSize);
        background-color: #0a89ff;
        user-drag: none;
        -webkit-user-drag: none;
        -moz-user-select: none;
      }

      .top-left {
        top: var(--offsetSize);
        left: var(--offsetSize);
        cursor: nw-resize;
      }

      .top-center {
        top: var(--offsetSize);
        left: 50%;
        transform: translateX(-50%);
        cursor: n-resize;
      }

      .top-right {
        top: var(--offsetSize);
        right: var(--offsetSize);
        cursor: ne-resize;
      }

      .center-right {
        top: 50%;
        right: var(--offsetSize);
        transform: translateY(-50%);
        cursor: e-resize;
      }

      .bottom-right {
        right: var(--offsetSize);
        bottom: var(--offsetSize);
        cursor: se-resize;
      }

      .bottom-center {
        bottom: var(--offsetSize);
        left: 50%;
        transform: translateX(-50%);
        cursor: s-resize;
      }

      .bottom-left {
        bottom: var(--offsetSize);
        left: var(--offsetSize);
        cursor: sw-resize;
      }

      .center-left {
        top: 50%;
        left: var(--offsetSize);
        transform: translateY(-50%);
        cursor: w-resize;
      }
    }
  }

  .embedded-comp:not(.interactable) {
    pointer-events: none;
    user-select: none;
    user-drag: none;
    -webkit-user-drag: none;
    -moz-user-select: none;
  }

  .slot {
    position: relative;
  }
}
</style>
