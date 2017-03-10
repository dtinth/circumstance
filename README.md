
# ![circumstance](http://i.imgur.com/oD4C88O.png)

[![NPM version][npm-svg]][npm]
[![Build status][travis-svg]][travis]
[![Code coverage][codecov-svg]][codecov]

[travis]: https://travis-ci.org/dtinth/circumstance
[travis-svg]: https://img.shields.io/travis/dtinth/circumstance.svg?style=flat
[npm]: https://www.npmjs.com/package/circumstance
[npm-svg]: https://img.shields.io/npm/v/circumstance.svg?style=flat
[codecov]: https://codecov.io/gh/dtinth/circumstance/src/master/README.md
[codecov-svg]: https://img.shields.io/codecov/c/github/dtinth/circumstance.svg

_Given-When-Then for your pure functions (e.g. Redux reducers)._

__circumstance__ lets you test your state-updating functions (including Redux reducers)
using the [Given-When-Then concept](http://martinfowler.com/bliki/GivenWhenThen.html).

> Note: This library is [generated from README.md](https://github.com/dtinth/essay). That’s why you don’t see any JavaScript file in this repository. What you’re reading is the library’s source code and tests.

## Example 1: A state updating function

Here’s a calculator test. See how natural the test looks!

```js
// examples/state/Calculator.test.js
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
// examples/state/Calculator.js
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
It contains few extra utility functions for testing reducers.
For example, here’s an example reducer:

```js
// examples/reducers/counter.js
export default function counterReducer (state = 0, action) {
  switch (action.type) {
    case 'INCREMENT': return state + 1
    case 'DECREMENT': return state - 1
  }
  return state
}
```

And here’s the corresponding test:

```js
// examples/reducers/counter.test.js
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
export { shouldContain } from './shouldContain'
export { withReducer } from './redux/withReducer'
export { state } from './state'
```


### `given(state) → Given`

Calling `given` with a `state` returns a `Given` object with these methods:

- `and(fn)` Apply `fn` to the state, e.g. to set up the scenario. Returns a `Given` object.
- `when(fn)` Apply `fn` to the state, e.g. to perform the action you intend to test. Returns a `When` object.
- `then(...fn, assertion)` Performs an assertion by calling `assertion` with `state` applied through `fn` successively from left to right. Returns a `Then` object. You can think of each `fn` as a selector. You can omit it, and the `assertion` will be called with the state.

```js
// given.js
import when from './_when'
import then from './_then'
export function given (state) {
  return {
    and: (fn) => given(fn(state)),
    when: (fn) => when(fn(state)),
    then: (...pipeline) => then(state)(...pipeline)
  }
}
export default given
```


#### The `When` object

A When object represents a scenario during the time where actions are performed. It has two methods:

- `and(fn)` Apply `fn` to the state, e.g. to perform more actions.
- `then(...fns, assertion)` Performs an assertion by calling `assertion` with `state` applied by `fns` successively from left to right. Returns a `Then` object.

```js
// _when.js
import then from './_then'
export function when (state) {
  return {
    and: (fn) => when(fn(state)),
    then: (...pipeline) => then(state)(...pipeline)
  }
}
export default when
```


#### The `Then` object

A Then represents a scenario after the actions are performed.

- `and(...fns, assertion)` Perform one more assertion. Returns a Then object.
- `when(fn)` Perform one more action. Returns a When object.

```js
// _then.js
import when from './_when'
export function then (state) {
  return (...pipeline) => (pipeline.reduce((x, f) => f(x), state), {
    and: (...nextPipeline) => then(state)(...nextPipeline),
    when: (g) => when(g(state))
  })
}
export default then
```

```js
// _then.test.js
import { given } from '.'
it('lets me assert the state directly', () =>
  given('hello')
  .when(state => state + '!')
  .then(state => assert(state.length === 6))
)
it('lets me assert a projection of the state', () =>
  given('hello')
  .when(state => state + '!')
  .then(state => state.length, length => assert(length === 6))
)

// Tip: Use functions to make your test look more natural!
import { shouldEqual } from '.'
import { property } from 'lodash'
it('lets me assert multiple times', () =>
  given({ x: 5, y: 20 })
  .then(property('x'), shouldEqual(5))
   .and(property('y'), shouldEqual(20))
)
```


### `withReducer(reducer)` (Redux)

Give it a Redux-style reducer, it will return two things:

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


### `shouldEqual(expectedValue)`

Give it an expected value, returns an assertion function that performs a deep comparison against the given value.

```js
// shouldEqual.js
import { deepEqual as assertDeepEqual } from 'assert'
export function shouldEqual (expectedValue) {
  return actualValue => assertDeepEqual(actualValue, expectedValue)
}
export default shouldEqual
```

### `shouldContain(expectedValue)`
Give it an expected value, returns function that perform comparison some keys of the given value.

```js
// shouldContain.js
import { deepEqual as assertDeepEqual } from 'assert'
import { keys, merge } from 'lodash'
export function shouldContain (expectedValue) {
  return actualValue => assertDeepEqual(
    keys(expectedValue).map((key) => ({ [key]: actualValue[key] })).reduce(merge, {}),
    expectedValue
  )
}
export default shouldContain
```

### `state(state)`

An identity function. For use with `.then()`. (See the test for example.)

```js
// state.js
export function state (currentState) {
  return currentState
}
export default state
```

```js
// state.test.js
import { given, state, shouldEqual, shouldContain } from '.'
it('lets me assert the state directly', () =>
  given('hello')
  .when(state => state.toUpperCase())
  .then(state, shouldEqual('HELLO'))
)

it('lets me assert the state contains some keys and values', () =>
  given({ x: 1, y: 2, z: 3 })
  .when(state => ({ ...state, x: state.x + 10 }))
  .then(state, shouldContain({ x: 11, z: 3 }))
)
```


## More examples

### Example 3: A Todos reducer

```js
// examples/reducers/todos.test.js
import { given, withReducer, state, shouldEqual } from '../..'
import todosReducer from './todos'
import { ADD_TODO } from '../constants/ActionTypes'
const { dispatch, initialState } = withReducer(todosReducer)

it('should start with a default todo item', () =>
  given(initialState)
  .then(state, shouldEqual([
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
  .then(state, shouldEqual([
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
  .then(state, shouldEqual([
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
