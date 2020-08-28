# Parallel generators

There should be a syntax `function **gen() { ... }` to create a parallel generator where you can resume from the same state multiple times. The first call would return an iterator result with an `undefined` value and `done: false`, starting at the first value. Each parallel iterator result contains the usual `done` and `value` properties, but also a `.next(value)` method to move to the next iteration and get the next state, a `.throw(value)` which works similarly, and a `.return(value)` that works similarly. Each result would also be iterable, where you can iterate to the last result.

Syntactically, it'd otherwise look like generators, aside from the extra asterisk after `function`, and the only runtime difference is that when you call `.next()` while it's still running, it just resumes from the last state. (This is the case for both sync and async generators.)

### Why?

Generators are really coroutines, and coroutines are really convenient for certain types of state machines. But sometimes, it's easier to model a state machine as a [decision tree](https://en.wikipedia.org/wiki/Decision_tree) of sorts. One concrete example of this is user input - [it's often much easier to model it as a user storyline rather than a state machine with a bunch of complex inputs](https://github.com/redux-saga/redux-saga).

### Example

Here's an example of how this might be used in practice, adapted from Redux Saga's readme:

```js
import Api from "..."

/*
 * Starts fetchUser on each dispatched `FETCH_USER` action.
 * This can happen concurrently, too.
 */
async function **reducer(model) {
    while (true) {
        const {type, ...action} = yield

        if (type === "FETCH_USER") {
            try {
                const user = await Api.fetchUser(action.userId)
                model.update({type: "USER_FETCH_SUCCEEDED", user: user})
            } catch (e) {
                model.update({type: "USER_FETCH_FAILED", message: e.message})
            }
        }
    }
}

export default reducer
```

The view might do something like this:

```js
import React, {useContext} from "react"

function View({userId}) {
    const reducer = useContext(Reducer)

    useEffect(() => {
        reducer.next({type: "FETCH_USER", userId})
    }, [userId])

    // render view
}
```
