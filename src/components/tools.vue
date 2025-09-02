<template>
  <div class="tools">
    <div class="mouse-tools">
      <button
        v-for="toolName in mouseTools"
        :key="toolName"
        class="tool"
        :class="{ active: isActive(toolName), [toolName]: true }"
        @click="selectTool(toolName)"
      >
        <div class="icon" />
      </button>
    </div>
    <tool-category
      name="All"
      :sub-tree="indexedTools"
      :active-tool="activeTool"
      @select-tool="selectTool"
    />
  </div>
</template>

<script>
import ToolCategory from './tool_category.vue';

const mouseTools = ['mouse', 'mover'];

export default {
  components: {
    ToolCategory
  },

  props: {
    stories: {
      type: Array,
      required: true
    },

    activeTool: {
      type: String,
      default: ''
    }
  },

  emits: ['select-tool', 'update-tools', 'mounted'],

  data: function () {
    return {
      mouseTools
    };
  },

  computed: {
    tools: function () {
      return Object.fromEntries(
        this.stories.flatMap(storyName =>
          Object.entries(storyName)
            .filter(([key]) => key !== 'default')
            .map(([key, val]) => [
              `${storyName.default.title}/${key}`,
              { ...val, definition: storyName.default, isStory: true }
            ])
        )
      );
    },

    indexedTools: function () {
      const dict = {};

      Object.entries(this.tools).forEach(([key, story]) => {
        const subKeys = key.split('/');
        this.indexStory(dict, subKeys, story);
      });

      return dict;
    }
  },

  mounted: function () {
    this.$emit('mounted', this.tools);
  },

  methods: {
    isActive: function (tool) {
      return this.activeTool === tool;
    },

    selectTool: function (toolName) {
      this.$emit('select-tool', toolName);
    },

    indexStory: function (dict, subKeys, story) {
      if (subKeys.length === 1) {
        dict[subKeys[0]] = story;
      } else {
        dict[subKeys[0]] ||= {};
        this.indexStory(dict[subKeys[0]], subKeys.splice(1), story);
      }
    }
  }
};
</script>

<style scoped>
.tools {
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  overflow-y: scroll;

  .mouse-tools {
    display: grid;
    grid-template-columns: repeat(2, 50px);
    grid-auto-rows: 50px;
  }

  :deep(.tool) {
    height: 50px;
    max-width: 200px;
    word-break: break-all;

    &.active {
      background-color: #fff;
    }

    .icon {
      width: 100%;
      height: 100%;
      background-position: center;
      background-repeat: no-repeat;
      background-size: 70%;
      display: flex;
      justify-content: center;
      align-items: center;
    }

    &.mouse .icon {
      background-image: url('../assets/mouse.svg');
    }

    &.mover .icon {
      background-image: url('../assets/mover.svg');
    }
  }
}
</style>
