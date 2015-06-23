![](/content/images/2015/06/shutterstock_262957136.png)

Flux has been with us for a while, and there are now countless frameworks based on the architecture. Some of them are just "syntactic sugar" while others depart significantly from the original idea. Personally I have never been a fan of any framework based on the Flux architecture, as its fundamental appeal lies in its combination of simplicity and robustness. There is no denying that it can be a bit verbose, but that's a minor complaint compared to its elegance and purity.

Nonetheless, there are a few annoyances to contend with, and we have been working for some time on adaptations designed to address them (inspired in large part by the [Om framework](https://github.com/omcljs/om)).

## Store dependencies
I believe this is the most unpleasant part of Flux. Facebook says that we should use `waitFor` to synchronise our action handlers, but they don't say anything about the root problem, which is dependencies between stores. In their [sample chat application](https://github.com/facebook/flux/blob/master/examples/flux-chat/js/stores/MessageStore.js#L100-L104), they seem absolutely fine with tight store interdependencies. In a small project like this, this is probably a viable option, but in bigger projects it can cause a huge mess in the codebase. 

There is also a subtle issue with those dependencies. Imagine you have two stores:

```javascript
class StoreA {
  onFooAction() {
    this.bar = this.mutateBar();
  }
}

class StoreB {
  onFooAction() {
    waitFor([StoreA.dispatchToken]);
    if (StoreA.bar === 'bar') {
      // Update view based on bar
    }
  }
}
```

There is a `Foo` action that both stores handle. StoreA mutates its internal state, while StoreB waits for the mutation then updates a view. So far, so good... right? The subtle problem occurs if we create a new action that mutates `bar`. It is far too easy to forget to handle that action in `StoreB`:

```javascript
class StoreA {

  onFooAction() {
    this.bar = this.mutateBar();
  }

  onBazAction() {
    this.bar = this.mutateBarDifferently();
  }
}
```
Bang! We've introduced a bug since our view is not consistent with the state. As you can see, besides turning your code into spaghetti, store interdependencies are also quite dangerous.

## Application code depends on the dispatcher
Let's assume for a moment that we don't need any store dependencies. Do you know what the purpose of the dispatcher is? Its most important concern is definitely coordinating actions, by which I mean using `waitFor`. But we are anticipating that there are no store interdependencies and therefore no need for `waitFor`.

In other words, if we found a way to get rid of those troublesome dependencies, we could theoretically forget about the dispatcher entirely. However, we don't want to do this, because the dispatcher is actually a great place for implementing replay, redo, undo, debugging, etc.

These super cool features are mostly not that super cool in the eyes of the customer, however, since they are much more service logic than domain logic. Ideally, we would like our stores and actions to be independent of the dispatcher. Imagine a store that is a pure function instead of a stateful class. Classes are in my opinion most useless part of ES6. We don't need them, and anyway we want to get rid of state, which is the root of much evil in software.

## Reducer aka stateless store for the win
Wait a minute? A stateless store? What the heck is that? No, you are not crazy, a store does not need to hold state. The main idea behind the stateless store is that you will keep all the data from all your stores in a single place; let's call it `ApplicationState`.

The biggest benefit is that you don't need dependencies between stores. The boundary of your stores used to be state, but now that the state is centralised, the boundaries between stores are much looser. You will also find that there is no longer any need for `waitFor`. If you still think there is, that probably means you need to reorganise your state tree to adapt it better to your requirements.

In the original Facebook implementation, stores had two concerns: to hold state and emit change events. We just separated the state out of the store, but we still need to notify views that changes have occurred. Now that we are treating our state as a single monolithic piece, we can emit one change for each update of our application state. This means that all views are notified whenever something has changed, and therefore we don't need `emitChange` for stores anymore.

So we have just removed state from stores and gotten rid of `emitChange` to boot. Do we still need to call it a "store"? No, we don't. I would even say we shouldn't, because it does not store anything. And this is where **reducer** comes to play. You might wonder why I call it a reducer. The answer is this snippet:

```javascript
  let applicationState = actions.reduce((state, action) => state, initialState);
```
A stateless store is nothing more than a `reduce` function of `actions` into `applicationState`.

## We want our components to be pure
What exactly is a **pure** component? A component is pure when, given the same props and state, the rendered result is the same. In other words, a pure component is not externally parametrised. In mathematical terms, the rendered markup is a function of properties and state.

Why do we want components to be pure? If the component is pure, with state and properties held in immutable data structures, the decision whether the component needs to be re-rendered is as simple as comparing the current state and properties with the new ones:

```javascript
  shouldComponentUpdate(nextProps, nextState) {
    return !shallowEqual(this.props, nextProps) ||
           !shallowEqual(this.state, nextState);
  }
```
You can read about the performance benefits of using immutable data structures in my [previous post](https://blog.javascripting.com/2015/03/31/turbocharge-your-react-application-with-shouldcomponentupdate-and-immutable-js/).

In practice, this means that your top-level component listens to all changes in your application state and passes the entire state tree down the component hierarchy to the corresponding components. The ES6 [`spread` operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_operator) is [essential](https://github.com/tomkis1/flux-boilerplate/blob/master/src/client/components/TodoList.jsx#L30). We don't even mind that our state tree emits just one change event for entire application, because our components are pure and we use immutable data structures. Therefore only stuff that has really changed is updated and everything is super fast.

## Independent actions and reducers
The only part of the Flux architecture that has a real implementation is the dispatcher. Ideally we would like to avoid mixing the dispatcher with our reducers and actions. As I mentioned, reducers are just pure `reduce` functions and they don't need to rely on the dispatcher. The same applies for actions. They don't need to know anything about the dispatcher. So the implementation might look like this:
```javascript
export default function addTodo(todo) {
  return {
    type: TODO_ADDED,
    payload: todo
  };
}

// or async

export default function addTodoAsync(todo) {
  return ApiService.insertTodo(() => {
    return {
      type: TODO_ADDED,
      payload: todo
    };
  });
}

export default function todoListReducer(action, state) {
  switch (action.type) {
    case TODO_ADDED:
      state = state.todos.add(action.payload);
    break;
  }

  return state;
}
```
As you can see there is no dependency on Flux. Actions and reducers shouldn't even be aware of the dispatcher. This conceptual approach makes unit testing really easy, because application domain logic is isolated and mocking is not necessary.

I have prepared a very simple [GitHub repository](https://github.com/tomkis1/flux-boilerplate) that illustrates the exact implementation details. Take a look in particular at `PureControllerView.jsx`, `TodoListReducer.js`, `CustomDispatcher.js` and `TodoActions.js`.