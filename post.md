# Composable Context API with Hooks

React's ContextAPI is a great light-weight alternative to using Redux for global state management.

Its important to understand that not every component will require the use of React's ContextAPI, or any global state management tool in general for the most part. Ideally, components should exist in a "functional" state-less fashion, for instance, not carrying any state, and instead, leveraging real-time values passed in through props.

For example:
```
    const UserNameDisplay = (props) => (<span>props.userName<span/>);
```
This state-less design enables for easier testing, and forces the logic and state to be kept in the component's parent. Essentially, keeping state centralized to prevent off-sync state within the app.

If we take for example a TODO app, more than likely we know we may need to keep a reference to the TODO items on the application at any one given time. This is because it enables the children from the main top-level component, say for example, the Todo component, from having to drill down the state of the `todos` down to each child component, and then each child that would require the `todos` would then in part need to drill down `todos` further down the chain.

For example (Not the right way of doing things):
```
    const SomeOtherChildComponent = ({todos}) => {
    		return (
    				<AnotherChildComponent todos={todos}/> // you get the idea by now ...
    		)
    }

    const TodosMainComponent = () => {
    	const todos = [];
    	return (
    			<SomeOtherChildComponent todos={todos}/>
    	)


    }
```
This is quite cumbersome. Prop drilling is perfectly fine if we are dealing with one level of component depth, however, when we have multiple levels of depths required, ContextAPI would provide for a better way of "Handing down" state to the child components of `TodosMainComponent`.

The idea is that we have a Provider component that sets up our state, and as many Consumer components to, well, consume that state.

Here's the gist:
```
    <SomeContext.Provider value={someState}>

    	<SomeComponent/>

    </SomeContext.Provider>
```
Ideally, we'd like a way to define our Context in a more "composable" way.

We can leverage the concept of React's Hooks to create a custom component that introduces a specific context state. In this case, a `todo` state. *[Stackblitz Todo Context Example](https://stackblitz.com/edit/react-todo-context-api)*

## Setting up Reducer and Initial State:

Let's start by first defining our reducer structure:
```
    import { HYDRATE_TODOS } from "./actionTypes";

    export const initialState = {
      todos: []
    };

    const reducer = (state = initalState, { type, payload }) => {
      switch (type) {
        case HYDRATE_TODOS:
          return { ...state, todos: payload };
        default:
          return state;
      }
    };

    export default reducer;
```
## Wiring up our Composed Provider Component:

We could have just defined the `todos` using the `useState` hook, since we are just dealing with an array of objects (single value), however, for the purposes of scaling this to also add in additional properties/actions to the state (Add, Remove, Update etc), we'll just start with a reducer.
```
    import React, { createContext, useReducer } from "react";
    import reducer, { initialState } from "./reducer"; // our reducer from above
```
The first thing we'd have to do is ensure that we are creating a React context
```
    import React, { createContext, useReducer } from "react";
    import reducer, { initialState } from "./reducer"; // our reducer from above

    export const TodosContext = createContext(); // our context for todos
```
Now, we can create a component that would accept other components as a "props" passed in. We can think of this component as the "Parent" component that will initialize our context and pass the context down to the children (the components passed in).
```
    import React, { createContext, useReducer } from "react";
    import reducer, { initialState } from "./reducer"; // our reducer from above

    export const TodosContext = createContext(); // our context for todos

    export const TodosProvider = ({ children }) => {
      const [state, dispatch] = useReducer(reducer, initialState); // intialize our reducer
      const value = [state, dispatch]; // what we'll expose to all children components

      return (
        <TodosContext.Provider value={value}>{children}</TodosContext.Provider>
      );
    };
```
Oh hey, look at that, we've essentially created a reusable component we can bring in to initialize our todo's context and pass in as many children as we'd like. This works in a similar fashion to that of React's Router. Where you have the main router component and the child routes nested underneath:
```
    <Router>
    	<Route/>
    	<Route/>
    </Router>
```
Its important to understand that we are essentially exposing the `state` and `dispatch` properties to all our child components. This would essentially allow our child components to alter the `todo` state by dispatching actions to our `todos` reducer, and also to read in our `todos` by using the `state` prop.

That's essentially all we need in terms of context scaffolding setup. Let's use it!

## Using the TODO Provider Component:

In our example case from above, we'll refactor the `TodosMainComponent` and its `ChildComponent` to display the list of TODOs using our new `TodoContext`:
```
    import React, { useContext, useEffect, Fragment } from 'react';
    import { TodoProvider, TodoContext } from './todos/contexts/TodoContext' // import our context provider
    import { HYDRATE_TODOS } from "./actionTypes";

    const TodoApp = () => {
    	return(
    			<Fragment>
    				<TodoProvider> //remember, we've already setup this provider with the value and initial state
    					<TodosMainComponent/>
    				</TodoProvider>
    			</Fragment>
    	)
    }

    const SomeOtherChildComponent = () => {
    		const [{todos}, todoDispatch] = useContext(TodoContext); // we can dispatch events or leverage the todo state here

    		const displayItems = (todos) => todos.map(todo =>
          <li key={todo.id.toString()}>{todo.body}</li>
      );

    	  return (
    	    <ul>{displayItems(todos)}</ul>
    	  )
    }

    const TodosMainComponent = () => {
    	const someTodoList = [{id: 1, body: 'Some todo'}];
    	const [{ todos }, todosDispatch] = useContext(TodoContext);

    	useEffect(()=> {
    			todoDispatch({type: HYDRATE_TODOS, payload: someTodoList});
    	}, []);

    	return (
    				<SomeOtherChildComponent/>
    	)
    }
```

## Conclusion

Obviously, this is a very simple example of the concepts, however, in real practice, it may be more suitable to wrap a set of routes in a particular context. For instance, you could do something like this:
```
    <TodoProvider>
            <Route path="/" exact component={TodoMainComponent} />
            <Route path="/todos/add" exact component={Add} />
    </TodoProvider>
```
This would allow you to insert todo's into your state from your Add component, and avoid having to go back to your backend to refresh the local state data.

We also need to keep in mind that React will eagerly re-render your components given any state change. So if you have a really large sub-tree of child components nested under one context, it may be worth looking into splitting your state and thus having multiple context's with a smaller child component set.

[Kent C Dodds also proposes an alternative solution to solving the performance issues introduced by complex, fast changing context values.](https://kentcdodds.com/blog/how-to-optimize-your-context-value) The idea here is that we'd split our actual state into its own provider, and our reducer dispatch function into another provider. Enabling only the components that are reading the `todo` state to render, but not any component that only alters the state. This may be a great solution if you have functional components such as buttons, menu displays, navigation footers etc.

If you are more interested in other solutions to improving React's Context API performance on large subtrees, [check Dan Abramov's proposed solutions on this.](https://github.com/facebook/react/issues/15156#issuecomment-474590693)

## Resources

*Inspiration for this post was drawn from Eduardo Robelos's post on [React Hooks: How to create and update Context.Provider](https://dev.to/oieduardorabelo/react-hooks-how-to-create-and-update-contextprovider-1f68)*