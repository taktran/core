# TSERS

**T**ransducer-**S**ignal-**E**xecutor framework for **R**eactive **S**treams.

[![Travis Build](https://img.shields.io/travis/tsers-js/core/master.svg?style=flat-square)](https://travis-ci.org/tsers-js/core)
[![Code Coverage](https://img.shields.io/codecov/c/github/tsers-js/core/master.svg?style=flat-square)](https://codecov.io/github/tsers-js/core)
[![NPM version](https://img.shields.io/npm/v/@tsers/core.svg?style=flat-square)](https://www.npmjs.com/package/@tsers/core)
[![Gitter](https://img.shields.io/gitter/room/tsers-js/chat.js.svg?style=flat-square)](https://gitter.im/tsers-js/chat)

> "tsers!"


## Motivation

*What if your application was just a pure function?* That's a very interesting
idea introduced by [Cycle.js](http://cycle.js.org/). Although the goal is noble,
the actual implementation of Cycle is not: the framework is designed to create
a "cycle" around the application. Yet, developers must implement their own 
"sub-cycles" by using Subjects and "isolate" their pure functions. 

Despite its poor implementation, Cycle has some great concepts. Maintaining 
those concepts and implementing them properly is the goal of **TSERS**:

* `main` is just a signal transducer `input$ => output$`
* Side-effects are wrapped with drivers
* `Model-View-Intent` instead of `Intent-Model-View`. Always.

## Hello world

The mandatory "Hello world" written with TSERS:
```javascript
import {Observable as O} from "rx"
import TSERS as "@tsers/core"
import makeReactDOM as "@tsers/react"

const main = T => in$ => {
  const {DOM: {h, withEvents, events}, decompose, compose} = T

  const [actions] = decompose(in$, "add$")
  return intent(view(model(actions)))

  function model({addBang$}) {
    const msg$ = addBang$
      .map(() => "!")
      .startWith("Tsers")
      .scan((acc, s) => acc + s)
    return msg$
  }

  function view(msg$) {
    const vdom$ = msg$.map(msg =>
      h("div", [
        h("h1", msg),
        h("button.add", "Click me!")
      ]))
    return withEvents(vdom$)
  }

  function intent(vdom$) {
    const addBang$ = events(vdom$, ".add", "click")
    const loop$ = compose({addBang$})
    const out$ = compose({DOM: vdom$})
    return [out$, loop$]
  }
}

const [Transducers, signals, execute] = TSERS({
  DOM: makeReactDOM("#app")
})
const { run } = Transducers
execute(run(signals, main(Transducers)))
``` 


## What is different compared to Cycle?

TSERS makes a clear distinction between signals and transducers:

Cycle's `main` is:
```javascript
const main = sources => sinks
``` 
Where `sources` can be either streams or transducers or stream generators or 
combination of those and `sinks` are streams of signals going to the "outside world".

TSERS' `main` is:
```javascript
const main = Transducers => input$ => output$
``` 
Where `Transducers` is always a collection of signal transducer functions, 
`input$` is a stream of events coming from the "outside world" and `output$` is a
stream of signals going to the "outside world".

And same applies for drivers. Cycle driver is a function:
```javascript
function driver(sink$) {
  return sources
}
```
Where `sink$` is a stream of signals coming from sinks and sources can be
either streams or transducers or stream generators or combination of those.

TSERS' `main` driver is a function:
```javascript
function driver() {
  return { signals, Transducers, executor: output$ => {} }
}
```  
Where `signals` is a stream of signals coming from the "outside world", 
`Transducers` is a collection of transducer functions and `executor` is
an interpreter that subscribes to the `output$` signals and creates 
side-effects based on those signals.


## Usage

TSERS provides only one public function via `default` exports. That function takes 
an object of drivers (see detailed driver spec below) and returns an array 
containing `Transducers`, `signals` and `execute`.

```javascript
import TSERS from "@tsers/core"
import makeReactDOM from "@tsers/react"
import main from "./your-app"

const [Transducers, signals, execute] = TSERS({
  DOM: makeReactDOM("#app")
})
``` 

In order to run your application, you must understand the role of those
`Transducers`, `signals` and `execute`.

### Signals

Signals is just a stream of events coming from the "outside world". These events
can be anything: user keyboard clicks, mouse movements, messages from WebSocket,
sounds from guitar pedals etc etc.

The values of signals stream are just `{key, val}` objects where `val` contains 
the signal data and `key` is the name of the driver that emitted the signal
(e.g. `DOM`). It's up to driver's implementation to decide whether it emits
signals or not - some drivers might emit them (like web-socket driver), other 
drivers might not.

### Transducers 

Transducers are the "switch army knife" that actually processes the input signals
to output signals. Don't get distracted by the name: [transducer](https://en.wikipedia.org/wiki/Transducer) 
is just a function that transforms signals `a` to `b`:
```
Transducer :: a$ => b$
```

As you can notice, `main` is actually just another signal transducer. 
Observable's `map`, `filter` and `flatMap` (for example) are also transducers.

`Transducers` from `TSERS` function is a JSON object that contains all transducers
from drivers, grouped by drivers name (e.g. if you have a `DOM` driver then you
have for example the `Transducers.DOM.events` transducer).

TSERS provides also a small set of built-in transducers for common tasks. 
The most important ones are: `decompose`, `run` and `compose`. 

#### `decompose :: (in$, ...keys) => [{[key]: [signals-of-key]}, rest$]`

As told before, input signals are just a stream of key-value pairs.`decompose` is 
a helper function meant to "extract" specific input signals from the rest. 

![decompose](doc/decompose.png)

```javascript
const input$ = O.of({key: "Foo", val: "foo!"}, {key: "Bar", val: "bar"}, {key: "Foo", val: "foo?"}, {key: "lol", val: "bal"})
const [decomposed, rest$] = decompose(input$, "Foo", "Bar")
decomposed.Foo.subscribe(::console.log)   // => "foo!", "foo?"
decomposed.Bar.subscribe(::console.log)   // => "bar"
rest$.subscribe(::console.log)            // =>  {key: "lol", val: "bal"}
```

#### `compose :: ({[key]: [signals-of-key]}, rest$ = O.never()) => output$`

`compose` does exactly opposite as `decompose` - it maps the given input values to
the `{key,val}` pairs based on the input template and merges them. For convenience, it also
takes rest input signals (key-value pairs) as a second (optional) argument and
merges them to the final output stream.

![compose](doc/compose.png)
```javascript
const foo$ = O.just("foo!")
const bar$ = O.just("bar..")
const rest$ = O.just({key: "lol", value: "bal"})
const out$ = compose({Foo: foo$, Bar: bar$}, rest$)
out$.subscribe(::console.log)   // => {key: "Foo", val: "foo!"}, {key: "Bar", val: "bar.."}, {key: "lol", val: "bal"}
```

Also note that `compose` and `decompose` are transitive:
```javascript
const input$ = ...
const keys = [ ... ]
const output$ = compose(...decompose(input$, ...keys))
// output$ and input$ streams produce same values
```

#### `run :: (input$, (input$ => [output$, loop$]) => output$`

`run` is the way to loop signals from downstream back to upstream. It takes
input signals and a transducer function producing `output$` and `loop$` signals
array - `output$` signals are passed through as they are, but `loop$` signals
are merged back to the transducer function as input signals.

Before `loop$` signals are merged to the other input signals, they are "masked"
with (`ext=false`) key. This ensures that `loop$` signals are "private": parent's 
`loop$` signals are not visible in child's `input$`.

`run` accepts the return value in many formats: you can omit the second array
element and return only `[output$]` without `loop$` signals (equivalent to
`[output$, O.never()]`. You can also return plain `output$` stream which is equivalent
to `[output$]`.


![run](doc/run.png)
```javascript
const input$ = compose({Foo: O.just("tsers")}, O.never(), true)
const main = input$ => {
  const [{Bar: bar$, Foo: foo$}] = decompose(input$, "Bar", "Foo")
  const output$ = bar$.map(x => x + "!")
  const loop$ = compose({Bar: foo$.map(x => x + "?")})
  return [output$, loop$]
}

const output$ = run(input$, main)
output$.subscribe(::console.log)  // => "tsers?!"
```


### Execute

`execute` is like Cycle's `run` but it doesn't make signal proxying from
`output$` back to `input$` (TSERS already has `Transducers.run` for it!). It's
only task is to subscribe to the output signals, interpret them and cause
the side-effects if necessary. 

`execute` is a composed function from drivers' executors so that each output
signal are routed to corresponding driver by using signal's `key` (so that it
matches with driver's key). 

```javascript
const main = T => in$ => {
  const {DOM: {h}, compose} = T
  const vdom$ = O.just(h("h1", "Tsers!"))
  return compose({DOM: vdom$})    // vdom$ signals are routed to "DOM" driver
}
```

`execute` returns a `dispose` function which can be called to unsubscribe ("stop")
drivers' executors:
```javascript
const dispose = execute(output$)
setTimeout(dispose, 1000)  // stop after 1 sec
```


### Running the app

Finally! But now it should be easy. We know that:

1. `signals :: input$`
2. `main :: Transducers => input$ => output$`
3. `Tranducers.run :: (input$, input$ => [output$, loop$]) => output$`
4. `execute :: output$ => dispose` 

Let's compose those:
```javascript
import TSERS from "@tsers/core"
import makeReactDOM from "@tsers/react"
import main from "./your-app"

const [Transducers, signals, execute] = TSERS({
  DOM: makeReactDOM("#app")
})
const { run } = Transducers

const dispose = execute(run(signals, main(Transducers)))
``` 

Now you may understand why the signature of `main` is `Transducers => input$ => output$`: 
it's all about composition. That's TSERS!


## Model-View-Intent

The one major difference between TSERS and Cycle is that TSERS implements the real `MVI` 
whereas Cycle implements `IMV`. In practice this means that in Cycle apps, the border
of **M**odel and **I**ntent becomes blurry when you have a cross-dependency between
them. The simplest case is a form validation:

1. In order to send the form to the server, you must have the form values (intent depends on model)
2. In order to show an AJAX spinner during the validation, the send status must be 
stored to the form (model depends on intent)

If course that is solvable also with `IMV` and there are more or less elegant solutions 
either leaking memory or not.

In `MVI` however, there is no exception - you can **always** apply `MVI` and loop
the model dependencies with `loop$` signals back to model.

```javascript
const main = T => in$ => {
  const {DOM, HTTP, decompose, compose} = T
  const [actions] = decompose(in$, "validate$", "validated$")
  return intent(view(model(actions)))
  
  function model({validate$, validated$}) {
    const form$ = validate$
      .map(showSpinner)
      .merge(validated$.map(embedValidationResultsAndRemoveSpinner))
      .startWith(initialValues)
      .shareReplay(1)
    return form$
  }
  
  function view(form$) {
    return [form$, buildFormVDOM(form$)]
  }
  
  function intent([form$, vdom$]) {
    const validate$ = DOM.events(vdom$, "button.validate", "click").withLatestFrom(form$)
    const validated$ = HTTP.req(validate$.map(toReqObject)).switch()
    const out$ = compose({DOM: vdom$, value$: form$})
    const loop$ = compose({validate$, validated$})
    return [out$, loop$]
  }
}

// index.js
execute(run(signals, main(Transducers)))
```

TODO: show how to implement Cycle with a few lines of TSERS

## Common Transducer API reference

TODO: examples

```javascript
//compose :: ({[key]: [signals-of-key]}, rest$ = O.never()) => output$
 
 
// decompose :: (in$, ...keys) => [{[key]: [signals-of-key]}, rest$]
 
 
// run :: (in$, (in$ => [out$, loop$])) => out$


// lift :: (out$$, ...keys) => [{[key]: [signals-of-key]}, rest$]
const out$$ = form$.map(f => run(in$, Child(Transducers, f.childValue)))
const [{DOM, value$}, rest$] = lift(out$$, "DOM", "value$")
 
// liftArray :: (outArr$, (val => out$), ...keys) => [{[key]: [signals-of-key]}, rest$]
const persons$ = form$.map(f => f.persons)
const [{DOM, value$}, rest$] = liftArray(persons$, person => run(in$, Person(Transducers, person)),
  "DOM", "value$")
// DOM is now an array of latest DOM values of Person components
// value$ is now an array of latest values of Persons
```


## License

MIT

