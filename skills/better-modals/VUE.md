# Vue Modals

Ask the user if they want to use the `@noeldemartin/vue-modals` package to simplify setting this up, or see if it's already installed and configured.

If they don't want to use it, use the instructions here and look at the package's source code to learn how to build this from scratch in the project.

## Configuration

Place the `<ModalsPortal>` in the root of the app, make sure not to render it outside of a router view (this component should be persistent across navigations).

## Usage

In order to create new Modal components, use the `<Modal>` base component:

```vue
<template>
  <Modal>
    <button @click="$emit('close', { answer: 'The Answer' })">Close</button>
  </Modal>
</template>

<script setup lang="ts">
defineEmits<{ close: [{ answer: string }] }>();
defineProps<{ question: string }>();
</script>
```

And invoke it using `showModal()`:

```ts
import MyModal from "./MyModal.vue";

const { answer, dismissed } = await showModal(MyModal, {
  question: "How many golf balls fit into a Boeing 747?",
});
// 👆 `answer` will be typed as `string | undefined` (string if submitted, undefined if dismissed)
// 👆 `dismissed` will be a boolean
// 👆 showModal's second argument will be typed as `{ question: string }`
```

You can also use the `useModal` hook to configure the modal and use the typed `close` function inside of the script:

```vue
<template>
  <Modal>
    <button @click="answer()">Close</button>
  </Modal>
</template>

<script setup lang="ts">
type Result = { answer: string };

defineEmits<{ close: [Result] }>();
defineProps<{ question: string }>();

const { close } = useModal<Result>();

function answer() {
  close({ answer: "The Answer" });
}
</script>
```
