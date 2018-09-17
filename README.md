# Epeli's Redux Stack for TypeScript

Fairly opinionated Redux Stack for TypeScript. This is made two design goals in mind:

1.  Be type safe
2.  Be terse (Redux doesn't have to be verbose!)

I really don't recommend you to use this as is because this is fairly living library but if you like something here feel free to fork this or copy paste some parts to your project.

If you happen to work with me, you're in luck because you are now working with something that is actually somewhat documented :)

## Install

    npm install @epeli/redux-stack

And you will need the React and Redux stuff

    npm install redux react-redux react @types/react @types/react @types/react-redux

## Exported functions

Small overview. See usage example in the end.

### `configureStore(options: Object): ReduxStore`

This basically a fork from [`@acemarke/redux-starter-kit`][starter] which is adapted to TypeScript.

Simplifies store creation. Adds redux-thunk middleware and creates devtools connection automatically.

[starter]: https://github.com/markerikson/redux-starter-kit

options:

-   `reducer?: Reducer`: Single reducer
-   `reducers?: Reducer[]`: Multiple reducers for the same state
-   `middleware?: Middleware[]`: Redux middlewares. By default add redux-thunk
-   `devTools?: boolean`: Enables or disables redux-devtools. By default is enabled
-   `preloadedState?: State`: Preload store with a state
-   `enhancers?: Enhancers[]`: Redux enhancers

### `createSimpleActions(actions: Object, options?: Object): SimpleActions`

Create action types, action creators and reducers in one go. Immutable updates are made type safe and terse with [Immer][].

[immer]: https://github.com/mweststrate/immer

This is originally forked from [wkrueger/redutser][redutser]. Huge props for creating it!

[redutser]: https://github.com/wkrueger/redutser

options:

-   `immer: boolean`: Set to `false` to disable immer usage
-   `actionTypePrefix: string`: Custom prefix for the generated actions types

### `createReducer(actions: SimpleActions)`

Create reducer from simple actions for the redux store

### `createThunks(actions: SimpleActions, thunks: Object)`

Create thunks from simple actions for side effects (api calls etc.).

## Usage example

```tsx
import {
    createSimpleActions,
    createReducer,
    createThunks,
    configureStore,
} from "@epeli/redux-stack";

/**
 * Define state as a single interface
 * */
interface State {
    count: number;
}

const initialState = {
    count: 0,
};

/**
 * Simple actions are simple. No side effects. Just state
 * updates using Immer. Everything else should be made
 * using thunks.
 */
const SimpleActions = createSimpleActions(initialState, {
    /**
     * draftState is an Immer proxy so the updates can be made in
     * mutable style but the actual result will be properly immutable
     */
    setCount(draftState, action: {newCount: number}) {
        draftState.count = action.newCount;
        return draftState;
    },

    /**
     * Actions without any params still need an empty action object
     */
    increment(draftState, action: {}) {
        draftState.count += 1;
        return draftState;
    },
});

/**
 * createThunks() infers state type and action types from SimpleActions
 */
const Thunks = createThunks(SimpleActions, {
    /**
     * Side effects should be created in thunks.
     * For example calling random() is a side effect.
     */
    setRandomCount() {
        return (dispatch, getState) => {
            dispatch(SimpleActions.setCount({newCount: Math.random()}));
        };
    },

    /**
     * Async operations such as network requests are side effects.
     *
     * Notice that the returned thunk is an async function!
     */
    fetchCount() {
        return async (dispatch, getState) => {
            const response = await request(API_URL);

            dispatch(
                SimpleActions.setCount({
                    newCount: response.body.count,
                }),
            );
        };
    },

    /**
     * Thunks can dispatch other thunks and await on them
     * if needed.
     */
    doubleFetch() {
        return async (dispatch, getState) => {
            // First fetch the count to the store state
            await dispatch(this.fetchCount());

            // and after that double it
            dispatch(
                SimpleActions.setCount({
                    newCount: getState().count * 2,
                }),
            );
        };
    },
});

const store = configureStore({
    // reducers option takes an array of reducers which all receive the same state object.
    reducers: [
        // Create reducer from the simple actions
        createReducer(SimpleActions),

        // If you need to keep your old reducers still around
        oldReducer,
    ],
});
// For other options use your editor intellisense or checkout the code.
```

## Usage with redux-render-prop

[`redux-render-prop`][rrp] is another library by me but it's bit more stable so it lives in it's own package.

```tsx
import {makeCreator} from "redux-render-prop";
import {bindActionCreators} from "redux";

const AllActions = {...SimpleActions, ...Thunks};

export const createMyAppComponent = makeCreator({
    prepareState: (state: State) => state,

    prepareActions: dispatch => {
        return bindActionCreators(AllActions, dispatch);
    },
});
```

For more comprehensive example checkout the `redux-render-prop` readme.

[rrp]: https://github.com/epeli/redux-render-prop
