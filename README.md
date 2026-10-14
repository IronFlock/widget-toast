# \<widget-toast>

A transient toast / notification widget for the IronFlock dashboard. The backend publishes a
message into a data-backend table; bind the message and severity columns of that table to this
widget and each newly published row pops up a short-lived toast (success / error / info /
warning) inside the widget tile.

This webcomponent follows the [open-wc](https://github.com/open-wc/open-wc) recommendation.

## Installation

```bash
npm i @ironflock/widget-toast
```

## Usage

```html
<script type="module">
  import '@ironflock/widget-toast/dist/widget-toast.js'
</script>

<!-- the tag is versioned (see package.json) so multiple versions can coexist -->
<widget-toast-1.0.0></widget-toast-1.0.0>
```

## Expected data format

```ts
interface ToastNotificationConfiguration {
  message?: string // data-driven: the latest published message (empty is ignored)
  type?: 'success' | 'error' | 'info' | 'warning' // data-driven severity (default 'info')
  displayTime?: number // ms before auto-dismiss; 0 = persist until closed (default 4000)
  position?:
    | 'top-right'
    | 'top-left'
    | 'top-center'
    | 'bottom-right'
    | 'bottom-left'
    | 'bottom-center' // default 'top-right'
  maxVisible?: number // cap on simultaneous toasts (default 3)
  showIcon?: boolean // default true
  showCloseButton?: boolean // default true (forced on when displayTime is 0)
  successColor?: string // optional accent override
  errorColor?: string
  infoColor?: string
  warningColor?: string
}
```

`message` and `type` are data-driven (bound to backend columns); everything else is static
configuration. See `src/definition-schema.json` for the authoritative schema and `CLAUDE.md`
for the rendering pipeline.

## Local Demo

```bash
npm install
npm start
```

Opens `http://localhost:8000/demo/` — the demo cycles sample messages every ~1.5s to show
stacking and auto-dismissal.
