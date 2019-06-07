# Keeping the UI in sync with the state

On of the most complex challenge web building an application is to make sure the UI is in sync with the data. For simple widget, pure HTML and javascript can be sufficient. However as your application grows complexity reflecting the data updates back to the DOM (and vice-versa) becomes extremely hard and makes the overall application fragile.

That's where UI frameworks comes to the rescue. Framework makes developers' life easier by abstracting away the rendering details and automatically updates the DOM as the component state change. When it comes to reflect state change to the DOM, we can divide the work that need to be done by javascript framework in 2 different parts: detecting that a component state has changed and updating the UI accordingly.

In this article we will focus on the first aspect, and what the different ways to implement change detection.

## Disclaimer

Educational content, should not be used in production code.
Highly simplified, focus on the high-level abstract and not on the internal details

## Component model

Will use raw web component in the rest for the rest of the article.
Need to define an abstraction on top of the web components.

```js
class Component extends HTMLElement {
  constructor() {
    super();

    this.attachShadow({
      mode: "open"
    });

    this._scheduleForRendering();
  }

  _scheduleForRendering() {
    if (!this._willRender) {
      this._willRender = true;

      Promise.resolve().then(() => {
        this._willRender = false;
        this.render();
      });
    }
  }

  render() {}
}
```

Let's create a base `Component` class to would hold the rendering and update logic. The class extends from `HTMLElement` because all the web components inherit from the `HTMLElement`. 

* `constructor`: attach the shadow root and schedule the component for rendering.
* `_scheduleForRendering`: if the component is not already scheduled for rendering, the `_willRender` flag is set and invoke the `_invokeRender` method in the next micro task.
* `render`: is empty on purpose, it need to be overriden in the child class.

Simple example of how to use this

```js
class XCounter extends Component {

}
```

