# Immutable generators

There should be a syntax `function **gen() { ... }` to create an immutable generator where you can resume from the same state multiple times. The first call would return an iterator result with an `undefined` value and `done: false`, starting at the first value. Each immutable iterator result contains the usual `done` and `value` properties, but also a `.next(value)` method to move to the next iteration and get the next state, a `.throw(value)` which works similarly, and a `.return(value)` that works similarly. Each result would also be iterable, where you can iterate to the last result.

Syntactically, it'd otherwise look like generators, aside from the extra asterisk after `function`.

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
async function **mySaga(send) {
    const {type, ...action} = yield

    if (type === "USER_FETCH_REQUESTED") {
        try {
            const user = await Api.fetchUser(action.payload.userId)
            send({type: "USER_FETCH_SUCCEEDED", user: user})
        } catch (e) {
            send({type: "USER_FETCH_FAILED", message: e.message})
        }
    }
}

/*
 * Does not allow concurrent fetches of user. If "USER_FETCH_REQUESTED" gets
 * dispatched while a fetch is already pending, that pending fetch is cancelled
 * and only the latest one will be run.
 */
async function **mySaga(send) {
    const {type, ...action} = yield {last: 1}

    if (type === "USER_FETCH_REQUESTED") {
        try {
            const user = await Api.fetchUser(action.payload.userId)
            send({type: "USER_FETCH_SUCCEEDED", user: user})
        } catch (e) {
            send({type: "USER_FETCH_FAILED", message: e.message})
        }
    }
}

export default mySaga
```

You also might choose to make virtual DOM components out of this, unnesting a lot of logic in the process and using the language for things that would ordinarily require explicit user hooks, like for error handling.

```js
function **MyComponent() {
    // Set up some component-local state
    let cleanup

    // Get the attributes, return early if nothing changed.
    const attrs = yield "attrs"
    if (!attrsChanged(attrs)) return cleanup

    try {
        // Render a tree
        const {elem} = yield h("div", {ref: "elem"}, [
            "some tree using ", attrs.value
        ])

        // And mutate it after it's been committed
        elem.classList.add("fancy")

        // Do something on cleanup
        return cleanup = () => elem.classList.remove("fancy")
    } catch (e) {
        // Catch an error thrown either here or in a child
    }
}
```

I'm leaving the drivers for each as an exercise for the reader, but trust me in that they aren't that complicated to construct. It's like generators, but you call `nextResult = prevResult.next(value)` instead of `nextResult = iter.next(value)`. (Well, maybe the virtual DOM example might be a little difficult, but it's not *too* difficult.)

### Potential FAQs

Q: Isn't this basically continuations?
A: Yes in that it's equivalent to them. Really, they operate somewhere between them and algebraic effects, and combined with an appropriate driver, could technically emulate them pretty easily. But yes, `gen.fork()` is not unlike saving a continuation to fire later.

Q: Why a subtype? Why not just add this to all generators?
A: Not all generators *can* be safely split like this. Many built-in ones like array iterators *could*, but some, like async iterators firing network requests, really *shouldn't* be splittable. This arose from some initial feedback.

Q: Why not offer a `tee` method and have generators (or a subtype thereof) override it?
A: Because that implies a buffer, which is *not* what this proposal is shooting for - it's specifically looking to replay the effects wholesale, with external state potentially having changed in the meantime.
