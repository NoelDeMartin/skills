---
name: better-modals
description: Use when creating or refactoring modals and overlays in frontend code (alerts, dialogs, popups, etc.). Use an imperative, promise-based architecture. Avoid creating inline modals.
---

Render modals and overlays using an imperative, promise-based architecture. They should use a global store to coordinate open modals, and render all of them in the same root location. Do not use declarative, state-driven inline components (e.g., `<MyModal :open="show" />`).

## Global infrastructure

Before creating any new infrastructure, check if the project already has one.

If it doesn't, it will need the following:

- A global store holding all the open modals. This store should have a unique id for each modal, and keep a reference to related data such as component or render function, input properties, and event listeners (for promise callbacks).
- A central location at the root of the app to render the modals. This component will use the global store to render all the open modals, using the proper stacking order and styles.
- A helper function to open a modal, which appends the modal to the global store and returns a Promise to be resolved when the modal is closed. This mechanism will use the listeners from the global store. Make sure that the Promise result considers both a successful submission of the modal (if it was a from with some result), and a dismissal.
- A reusable Modal component. This component will encapsulate all the styles and behavior to interact with the global infrastructure, and individual modals will re-use it. For example, this component could be a wrapper for the native `<dialog>` element.

Example of naming for these components and helpers:

- Root location: `<ModalsPortal>`.
- Helper function to open modals: `showModal()`.
- Base class: `<Modal>`.

## Framework Guides

- If the application is built using Vue, check out [VUE.md](./VUE.md).
