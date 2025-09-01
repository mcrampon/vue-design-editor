<template>
  <div class="title" @click="expanded = !expanded">
    <div>{{ name }}</div>
    <div v-if="expanded">^</div>
    <div v-else>v</div>
  </div>

  <div v-if="expanded" class="sub-tree">
    <template v-for="[key, value] in Object.entries(subTree)" :key="key">
      <button
        v-if="value.isStory"
        class="tool"
        :class="{ active: isActive(key), [key]: true }"
        @click="selectTool(key)"
      >
        {{ key }}
      </button>
      <tool-category
        v-else
        :name="key"
        :top-level="false"
        :sub-tree="value"
        :active-tool="activeTool"
        @select-tool="selectTool"
      />
    </template>
  </div>
</template>

<script>
export default {
  name: 'ToolCategory',

  props: {
    name: {
      type: String,
      required: true
    },

    subTree: {
      type: Object,
      required: true
    },

    activeTool: {
      type: String,
      default: ''
    },

    topLevel: {
      type: Boolean,
      default: true
    }
  },

  emits: ['select-tool'],

  data: function () {
    return {
      expanded: this.topLevel
    };
  },

  methods: {
    isActive: function (tool) {
      return this.activeTool === tool;
    },

    selectTool: function (toolName) {
      if (this.topLevel) {
        this.$emit('select-tool', toolName);
      } else {
        this.$emit('select-tool', `${this.name}/${toolName}`);
      }
    }
  }
};
</script>

<style scoped>
.title {
  display: flex;
  gap: 1rem;
  cursor: pointer;
}

.sub-tree {
  padding-left: 1rem;
  display: flex;
  flex-direction: column;
}
</style>
