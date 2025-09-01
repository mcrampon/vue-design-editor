# Vue Design Editor

Initially developed while working at [Sumdog](https://learn.sumdog.com).

## TODO

- add prettier + eslint + pre-commit hook
- add proper readme
- map component props to argtype controls
- add layer structure

## Usage

```js
import VueDesignEditor from 'vue-design-editor';
import 'vue-design-editor/dist/vue-design-editor.css';

// Import your stories
import * as myStory from 'my_story.stories.js';

// Render VueDesignEditor with your stories
<VueDesignEditor :stories="[myStory]" />
```

## Requirements

Your Vue app needs to use the full Vue build (compiler-included). The runtime-only build will not be able to render dynamic components.
