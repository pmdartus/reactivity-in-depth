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
            mode: 'open',
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

-   `constructor`: attach the shadow root and schedule the component for rendering.
-   `_scheduleForRendering`: if the component is not already scheduled for rendering, the `_willRender` flag is set and invoke the `_invokeRender` method in the next micro task.
-   `render`: is empty on purpose, it need to be overriden in the child class.

Let's create a todo list example using the base `Component` class. First we need to define the the different template elements: one template for the todo list and a template for each todo list item.

```html
<template id="todo-list">
    <style>
        span.completed {
            text-decoration: line-through;
        }
    </style>

    <input id="todo-input" />
    <ul></ul>
</template>

<template id="todo-item">
    <li>
        <input type="checkbox" />
        <span></span>
        <button>-</button>
    </li>
</template>
```

The todo list implementation is pretty naive at point, every class to `render` wipes out the entire shadow tree (via. `this.shadowRoot.innerHTML`) and rerender the entire todo list. Note, bad approach suffers from performance issues and poor user experience (focus loss).

```js
const todoListTmpl = document.querySelector('#todo-list');
const todoItemTmpl = document.querySelector('#todo-item');

class XTodoList extends Component {
    constructor() {
        super();

        this.nextId = 2;
        this.todos = [
            {
                id: 1,
                title: 'Learn reactivity',
                completed: false,
            },
        ];
    }

    render() {
        const items = this.todos.map(todo => {
            const { id, title, completed } = todo;

            const frag = todoItemTmpl.content.cloneNode(true);

            const titleSpan = frag.querySelector('span');
            titleSpan.textContent = title;
            if (completed) {
                titleSpan.classList.add('completed');
            }

            const completeCheckbox = frag.querySelector('input');
            completeCheckbox.checked = completed;
            completeCheckbox.addEventListener('click', () =>
                this.toggleCompleted(id),
            );

            const removeButton = frag.querySelector('button');
            removeButton.addEventListener('click', () =>
                this.handleRemoveTodo(id),
            );

            return frag;
        });

        const frag = todoListTmpl.content.cloneNode(true);

        const todoInput = frag.querySelector('input');
        todoInput.addEventListener('change', () =>
            this.handleAddTodo(todoInput.value),
        );

        const list = frag.querySelector('ul');
        list.append(...items);

        this.shadowRoot.innerHTML = '';
        this.shadowRoot.appendChild(frag);
    }

    handleAddTodo(title) {
        this.todos.push({
            id: this.nextId++,
            title,
            completed: false,
        });

        this._scheduleForRendering();
    }

    handleRemoveTodo(todoId) {
        this.todos = this.todos.filter(todo => {
            return todo.id !== todoId;
        });

        this._scheduleForRendering();
    }

    toggleCompleted(todoId) {
        this.todos = this.todos.map(todo => {
            return todo.id === todoId
                ? { ...todo, completed: !todo.completed }
                : todo;
        });

        this._scheduleForRendering();
    }
}

customElements.define('x-todo-list', XTodoList);
```

Need to invoke `this._scheduleForRendering` in order to reflect data updates back to the DOM: error prone.

## Immutable state

Some framework favors immutable state (like React). The `state` property in React is a special property used to store the local state of the component. From the [React official document](https://reactjs.org/docs/react-component.html#state):

> Never mutate `this.state` directly, as calling `setState()` afterwards may replace the mutation you made. Treat `this.state` as if it were immutable.

By invoking `setState`, React becomes aware of the state change and will schedule a new render invocation in the future.

Let see how to implement `setState` on our base `Component` class:

```js
class Component extends HTMLElement {
    constructor() {
        super();

        this.attachShadow({
            mode: 'open',
        });

        this.state = {};

        this._scheduleForRendering();
    }

    setState(newState) {
        this.state = newState;
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
