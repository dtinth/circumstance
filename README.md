
# circumstance

_BDD for your pure functions (e.g. Redux reducers)._

__circumstance__ lets you write your state-updating functions (including Redux reducers)
using Given-When-Then concept.

![Concept](http://i.imgur.com/W81SZzL.png)


## Example 1: A state updating function

Here’s a calculator test. See how natural the test looks!

```js
// example/state/Calculator.test.js
import { given, shouldEqual } from '../..'

import {
  initialState,
  textToDisplay,
  keyDigit,
  pressPlusButton,
  pressEqualButton
} from './Calculator'

it('should start with 0', () =>
  given(initialState)
  .then(textToDisplay, shouldEqual('0'))
)

it('should allow entering digits', () =>
  given(initialState)
  .when(keyDigit(1))
  .and(keyDigit(5))
  .then(textToDisplay, shouldEqual('15'))
)

it('should allow adding numbers', () =>
  given(initialState)
  .and(keyDigit(9))
  .when(pressPlusButton)
  .then(textToDisplay, shouldEqual('9'))
  .when(keyDigit(3))
  .then(textToDisplay, shouldEqual('3'))
  .when(keyDigit(3))
  .then(textToDisplay, shouldEqual('33'))
  .when(pressEqualButton)
  .then(textToDisplay, shouldEqual('42'))
)
```

And here’s the calculator state module. See how all functions are pure (although the calculator is still very incomplete and buggy!)

```js
// example/state/Calculator.js
export const initialState = { current: null, operand: 0 }

// Commands: they return functions that update states
export function keyDigit (digit) {
  return state => ({ ...state,
    current: (state.current || 0) * 10 + (+digit)
  })
}

export function pressPlusButton (state) {
  return { ...state,
    current: null,
    operand: state.current
  }
}

export function pressEqualButton (state) {
  return { ...state,
    current: state.current + state.operand,
    operand: null
  }
}

// Queries: they return functions that queries states
export function textToDisplay (state) {
  return `${state.current || state.operand || 0}`
}
```


## Example 2: A Redux reducer

`circumstance` can easily be used with Redux reducers.
For example, here’s an example reducer:

```js
// example/reducers/counter.js
export default function counterReducer (state = 0, action) {
  switch (action.type) {
    case 'INCREMENT': return state + 1
    case 'DECREMENT': return state - 1
  }
  return state
}
```

```js
// example/reducers/counter.test.js
import counterReducer from './counter'
import { given, withReducer, state, shouldEqual } from '../..'
const { dispatch, initialState } = withReducer(counterReducer)

it('should start with 0', () =>
  given(initialState)
  .then(state, shouldEqual(0))
)
it('should increment', () =>
  given(initialState)
  .when(dispatch({ type: 'INCREMENT' }))
  .then(state, shouldEqual(1))
)
it('should allow repeated increment', () =>
  given(initialState)
  .when(dispatch({ type: 'INCREMENT' }))
  .and(dispatch({ type: 'INCREMENT' }))
  .then(state, shouldEqual(2))
)
it('should decrement', () =>
  given(initialState)
  .when(dispatch({ type: 'DECREMENT' }))
  .then(state, shouldEqual(-1))
)
```


## API

This is the public API:

```js
// index.js
export { given } from './given'
export { shouldEqual } from './shouldEqual'
export { stateShouldEqual } from './stateShouldEqual'
export { withReducer } from './redux/withReducer'
export { state } from './state'
```


### `given(state) → Given`

Calling `given` with a `state` returns a `Given` object with these methods:

- `and(fn)` Apply `fn` to the state, e.g. to set up the scenario. Returns a `Given` object.
- `when(fn)` Apply `fn` to the state, e.g. to perform the action you intend to test. Returns a `When` object.
- `then([fn,] assertion)` Performs an assertion by calling `assertion` with `fn(state)`. Returns a `Then` object. You can think of `fn` as a selector. You can omit it, and the `assertion` will be called with the state.

```js
// given.js
import when from './_when'
import then from './_then'
export function given (state) {
  return {
    and: (fn) => given(fn(state)),
    when: (fn) => when(fn(state)),
    then: (fn, assertion) => then(state)(fn, assertion)
  }
}
export default given
```


#### The `When` object

A When object represents a scenario during the time where actions are performed. It has two methods:

- `and(fn)` Apply `fn` to the state, e.g. to perform more actions.
- `then([fn,] assertion)` Performs an assertion by calling `assertion` with `fn(state)`. Returns a `Then` object.

```js
// _when.js
import then from './_then'
export function when (state) {
  return {
    and: (fn) => when(fn(state)),
    then: (fn, assertion) => then(state)(fn, assertion)
  }
}
export default when
```


#### The `Then` object

A Then represents a scenario after the actions are performed.

- `and([fn,] assertion)` Perform one more assertion. Returns a Then object.
- `when(fn)` Perform one more action. Returns a When object.

```js
// _then.js
import when from './_when'
export function then (state) {
  return (fn, assertion) => (assertion ? assertion(fn(state)) : fn(state), {
    and: (fn, assertion) => then(state)(fn, assertion),
    when: (g) => when(g(state))
  })
}
export default then
```


### withReducer (Redux)

Give it a reducer, it will return two things:

- `initialState` The initial state of this reducer.
- `dispatch` A function that takes an action object, and returns a function to update the state as if the reducer is called using this action.

```js
// redux/withReducer.js
export function withReducer (reducer) {
  return {
    initialState: reducer(void 0, { type: '@@redux/INIT' }),
    dispatch: action => state => reducer(state, action)
  }
}
export default withReducer
```


### shouldEqual

A simple function that returns an assertion function, which checks if the state is equal to the expected state.

```js
// shouldEqual.js
import { deepEqual as assertDeepEqual } from 'assert'
export function shouldEqual (expectedValue) {
  return actualValue => assertDeepEqual(actualValue, expectedValue)
}
export default shouldEqual
```

### state

An identity function...

```js
// state.js
export function state (currentState) {
  return currentState
}
export default state
```



## More examples

## Example 3: A Todos reducer

```js
// examples/reducers/todos.test.js
import { given, withReducer, stateShouldEqual } from '../..'
import todosReducer from './todos'
import { ADD_TODO } from '../constants/ActionTypes'
const { dispatch, initialState } = withReducer(todosReducer)

it('should start with a default todo item', () =>
  given(initialState)
  .then(stateShouldEqual([
    {
      text: 'Use Redux',
      completed: false,
      id: 0
    }
  ]))
)

it('should handle ADD_TODO after empty state', () =>
  given([ ])
  .when(dispatch({
    type: ADD_TODO,
    text: 'Run the tests'
  }))
  .then(stateShouldEqual([
    {
      text: 'Run the tests',
      completed: false,
      id: 0
    }
  ]))
)

it('should handle more ADD_TODO', () =>
  given([
    {
      text: 'Use Redux',
      completed: false,
      id: 0
    }
  ])
  .when(dispatch({
    type: ADD_TODO,
    text: 'Run the tests'
  }))
  .then(stateShouldEqual([
    {
      text: 'Run the tests',
      completed: false,
      id: 1
    },
    {
      text: 'Use Redux',
      completed: false,
      id: 0
    }
  ]))
)
```

The todos reducer is [stolen from Redux docs](http://redux.js.org/docs/recipes/WritingTests.html):

```js
// examples/reducers/todos.js
import { ADD_TODO } from '../constants/ActionTypes'

const initialState = [
  {
    text: 'Use Redux',
    completed: false,
    id: 0
  }
]

export default function todos(state = initialState, action) {
  switch (action.type) {
    case ADD_TODO:
      return [
        {
          id: state.reduce((maxId, todo) => Math.max(todo.id, maxId), -1) + 1,
          completed: false,
          text: action.text
        },
        ...state
      ]

    default:
      return state
  }
}
```

And the action types…

```js
// examples/constants/ActionTypes.js
export const ADD_TODO = 'ADD_TODO'
```
