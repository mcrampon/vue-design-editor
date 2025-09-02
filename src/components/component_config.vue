<template>
  <div class="config">
    <div v-if="Object.keys(argTypes).length === 0">No controls</div>
    <div
      v-for="[argName, argType] in Object.entries(argTypes)"
      :key="argName"
      class="row"
    >
      <label :for="`${argName}-config`">{{ argName }}</label>

      <!-- eslint-disable vuejs-accessibility/form-control-has-label -->
      <div>
        <!--
          TODO: figure out automated mapping from prop types
          (cf. https://github.com/vue-styleguidist/vue-styleguidist/tree/dev/packages/vue-docgen-api)
        -->

        <!-- boolean -->

        <input
          v-if="controlType(argType) === 'boolean'"
          type="checkbox"
          :name="`${argName}-config`"
          :checked="component.props[argName]"
          @change="() => updateCheckbox(argName, e)"
        />

        <!-- object -->

        <JsonEditorVue
          v-if="
            controlType(argType) === 'object' ||
            controlType(argType) === 'array'
          "
          mode="text"
          :main-menu-bar="false"
          :navigation-bar="false"
          :status-bar="false"
          :model-value="
            component.props[argName] || defaultJsonValue(controlType(argType))
          "
          @update:model-value="val => updateJson(argName, val)"
        />

        <input
          v-if="controlType(argType) === 'file'"
          type="file"
          :name="`${argName}-config`"
          multiple=""
          accept="image/*"
          @change="e => updateFiles(argName, e)"
        />

        <!-- enum -->

        <div
          v-if="
            controlType(argType) === 'radio' ||
            controlType(argType) === 'inline-radio'
          "
          class="fieldset"
          :class="{ inline: controlType(argType) === 'inline-radio' }"
        >
          <div v-for="option in argType.options" :key="option">
            <input
              type="radio"
              :name="`${argName}-config`"
              :value="option"
              @change="e => updateProp(argName, e)"
            />
            <label :for="`${argName}-config`">{{ option }}</label>
          </div>
        </div>

        <div
          v-if="
            controlType(argType) === 'check' ||
            controlType(argType) === 'inline-check'
          "
          class="fieldset"
          :class="{ inline: controlType(argType) === 'inline-check' }"
        >
          <div v-for="option in argType.options" :key="option">
            <input
              type="checkbox"
              :name="`${argName}-config`"
              :value="option"
              @change="e => updateCheckboxes(argName, e)"
            />
            <label :for="`${argName}-config`">{{ option }}</label>
          </div>
        </div>

        <select
          v-if="
            controlType(argType) === 'select' ||
            controlType(argType) === 'multi-select'
          "
          :name="`${argName}-config`"
          :multiple="controlType(argType) === 'multi-select'"
          @change="e => updateProp(argName, e)"
        >
          <option
            v-for="option in argType.options"
            :key="option"
            :value="option"
            :selected="option === component.props[argName]"
          >
            {{ option }}
          </option>
        </select>

        <!-- number -->

        <input
          v-if="controlType(argType) === 'number'"
          :name="`${argName}-config`"
          type="number"
          :value="component.props[argName]"
          :min="argType.control.min"
          :max="argType.control.max"
          :step="argType.control.step"
          @change="e => updateProp(argName, e)"
        />

        <input
          v-if="controlType(argType) === 'range'"
          :name="`${argName}-config`"
          type="range"
          :value="component.props[argName]"
          :min="argType.control.min"
          :max="argType.control.max"
          :step="argType.control.step"
          @change="e => updateProp(argName, e)"
        />

        <!-- string -->

        <input
          v-if="controlType(argType) === 'color'"
          :name="`${argName}-config`"
          type="color"
          :value="component.props[argName]"
          @change="e => updateProp(argName, e)"
        />

        <input
          v-if="controlType(argType) === 'date'"
          :name="`${argName}-config`"
          type="date"
          :value="component.props[argName]"
          @change="e => updateProp(argName, e)"
        />

        <textarea
          v-if="controlType(argType) === 'text'"
          :name="`${argName}-config`"
          :value="component.props[argName]"
          @keyup="e => updateProp(argName, e)"
        />
      </div>
      <!-- eslint-enable vuejs-accessibility/form-control-has-label -->
    </div>
    <div
      v-for="availableSlot in component.canvasData.availableSlots"
      :key="availableSlot"
    >
      {{ availableSlot }} slot:
      <button @click="() => toggleSlot(availableSlot)">
        {{ toggleCta(availableSlot) }}
      </button>
      <button
        v-if="slotEnabled(availableSlot)"
        @click="() => editSlot(availableSlot)"
      >
        Edit
      </button>
    </div>
  </div>
</template>

<script>
import JsonEditorVue from 'json-editor-vue';

const defaultJsonValues = {
  array: () => [],
  object: () => ({})
};

export default {
  components: {
    JsonEditorVue
  },

  props: {
    component: {
      type: Object,
      required: true
    }
  },

  emits: ['update-prop', 'toggle-slot', 'edit-slot'],

  computed: {
    argTypes: function () {
      return {
        ...(this.component.canvasData.definition.argTypes || {}),
        ...(this.component.canvasData.argTypes || {})
      };
    }
  },

  methods: {
    updateJson: function (argName, val) {
      const parsedVal = JSON.parse(val);
      this.$emit('update-prop', argName, parsedVal);
    },

    defaultJsonValue: function (type) {
      return defaultJsonValues[type]();
    },

    updateFiles: async function (argName, e) {
      const files = e.target.files;

      const filePromises = Object.keys(files).map(key => {
        return new Promise(resolve => {
          const file = files[key];
          var reader = new FileReader();
          reader.onload = function () {
            resolve(reader.result);
          };
          reader.readAsDataURL(file);
        });
      });

      const dataFiles = await Promise.all(filePromises);

      this.$emit('update-prop', argName, dataFiles);
    },

    updateCheckboxes: function (argName, e) {
      const selected = Array.from(
        e.target.closest('.fieldset').querySelectorAll('input:checked')
      ).map(cb => cb.value);
      this.$emit('update-prop', argName, selected);
    },

    updateCheckbox: function (argName, e) {
      this.$emit('update-prop', argName, e.target.checked);
    },

    updateProp: function (argName, e) {
      this.$emit('update-prop', argName, e.target.value);
    },

    controlType: function (argType) {
      return argType.control.type || argType.control;
    },

    toggleSlot: function (availableSlot) {
      this.$emit('toggle-slot', availableSlot);
    },

    toggleCta: function (availableSlot) {
      if (this.slotEnabled(availableSlot)) {
        return 'Disable';
      } else {
        return 'Enable';
      }
    },

    slotEnabled: function (availableSlot) {
      return this.component.canvasData.slots?.[availableSlot];
    },

    editSlot: function (availableSlot) {
      this.$emit('edit-slot', availableSlot);
    }
  }
};
</script>

<style scoped>
.config {
  overflow: auto;
  flex-shrink: 0;
  color: #eee;
  background-color: #444;
  padding: 2rem;
  display: flex;
  flex-direction: column;
  gap: 2rem;
  box-shadow: -3px 0px 11px 5px rgba(0, 0, 0, 0.4);
  z-index: 1;
}

.inline {
  display: flex;
}

:deep(.jse-error) {
  display: none;
}
</style>
