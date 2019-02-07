# Forkable generators

I'm suggesting we add a `multi function *gen() { ... }` to allow a generator to be forked and entered from a state multiple times. This would return a generator with a `.fork()` method that returns a new forkable generator cloned from the existing generator's state.

### Why?

Generators are really coroutines, and coroutines are really convenient for certain types of state machines. But sometimes, it's easier to model a state machine as a [decision tree](https://en.wikipedia.org/wiki/Decision_tree) of sorts. One concrete example of this is user input - [it's often much easier to model it as a user storyline rather than a state machine with a bunch of complex inputs](https://github.com/redux-saga/redux-saga). It also makes it far easier to implement undo/redo, since you could just save and revert back to the previous generator state and update the view appropriately.

### Example

Here's an example of how this might be used in practice, adapted from Redux Saga's readme:

```js
import Api from '...'

/*
 * Starts fetchUser on each dispatched `USER_FETCH_REQUESTED` action.
 * Allows concurrent fetches of user.
 */
async multi function *mySaga() {
    const {type, ...action} = yield ["receive"]

    if (type === "USER_FETCH_REQUESTED") {
        try {
            const user = await Api.fetchUser(action.payload.userId)
            yield ["send", {type: "USER_FETCH_SUCCEEDED", user: user}]
        } catch (e) {
            yield ["send", {type: "USER_FETCH_FAILED", message: e.message}]
        }
    }
}

/*
 * Does not allow concurrent fetches of user. If "USER_FETCH_REQUESTED" gets
 * dispatched while a fetch is already pending, that pending fetch is cancelled
 * and only the latest one will be run.
 */
async multi function* mySaga() {
    const {type, ...action} = yield ["receive", {last: 1}]

    if (type === "USER_FETCH_REQUESTED") {
        try {
            const user = await Api.fetchUser(action.payload.userId)
            yield ["send", {type: "USER_FETCH_SUCCEEDED", user: user}]
        } catch (e) {
            yield ["send", {type: "USER_FETCH_FAILED", message: e.message}]
        }
    }
}

export default mySaga
```

From the consumer side, you could make a relatively simple driver for this, complete with undo/redo support:

```js
class Driver {
    constructor(gen, send) {
        this._states = []
        this._index = 0
        this._send = send
        this._start = (async () => {
            for await (const [type, params] of gen) {
                if (type === "receive") {
                    this._states[0] = {label: undefined, gen, limit: +params.limit, active: 1}
                    this._start = undefined
                    return
                }
                if (type === "send") (0, this._send)(params)
            }
        })()
    }
    async undo() {
        if (this._index !== 0) {
            await (0, this._states[this._index--].undo)()
        }
    },
    async redo() {
        if (this._index !== this._states.length) {
            await (0, this._states[++this._index].redo)()
        }
    },
    get labels() { return this._states.slice(1).map(state => state.label) },
    get current() { return this._states[this._index].label },
    get currentIndex() { return this._index - 1 },
    async dispatch(action) {
        if (this._start != null) await this._start
        const state = this._states[this._index]
        if (state.active === state.limit) return
        state.active++
        const gen = state.gen.fork()
        while (true) {
            const result = await gen.next(action)
            if (result.done) { state.active--; return }
            const [type, params] = result.value
            if (type === "receive") {
                states[++this._index] = {
                    label: action.type, gen,
                    limit: Math.min(limit, params.limit),
                    undo: params.undo, redo: params.undo,
                    active: 1,
                }
                states.length = this._index + 1
                return
            }
            if (type === "send") (0, this._send)(params)
        }
    }
}
```

### Potential FAQs

Q: Isn't this basically continuations?
A: Yes in that it's equivalent to them. Really, they operate somewhere between them and algebraic effects, and combined with an appropriate driver, could technically emulate them pretty easily. But yes, `gen.fork()` is not unlike saving a continuation to fire later.

Q: Why a subtype? Why not just add this to all generators?
A: Not all generators *can* be safely split like this. Many built-in ones like array iterators *could*, but some, like async iterators firing network requests, really *shouldn't* be splittable.

Q: Why not offer a `tee` method and have generators (or a subtype thereof) override it?
A: Because that implies a buffer, which is *not* what this proposal is shooting for - it's specifically looking to replay the effects wholesale, with external state potentially having changed in the meantime.
