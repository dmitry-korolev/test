
# Stapp

Stapp is the highly opinionated application state-management tool based on redux and RxJS with significantly reduced boilerplate. The primary goal of Stapp is to provide an easy way to create simple, robust and reusable applications.

Stapp comprises all the best practices and provides instruments to write understandable and predictable code.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
### Table of contents

- [Introduction](#introduction)
- [Core concepts](#core-concepts)
- [Getting started](#getting-started)
- [Modules](#modules)
  - [Modules: handling state](#modules-handling-state)
  - [Modules: events](#modules-events)
    - [User interactions](#user-interactions)
    - [Modules logic](#modules-logic)
    - [Testing](#testing)
  - [Modules: epics](#modules-epics)
  - [Modules: module factories](#modules-module-factories)
  - [Modules: higher-order module factories](#modules-higher-order-module-factories)
- [API](#api)
  - [Core](#core)
  - [Epics utilities](#epics-utilities)
  - [Core events](#core-events)
  - [Modules](#modules-1)
    - [FormBase](#formbase)
    - [Persist](#persist)
  - [React utilities](#react-utilities)
- [Peer dependencies](#peer-dependencies)
- [License](#license)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

Some obvious statements:

1. Small applications are easier to develop and maintain.
2. Small blocks of business logic are easier to develop and maintain.
3. Applications tend to become larger and harder to develop and maintain.

Solution: separating applications into small and independent microapps.

## Core concepts

1. The application  consists of several logical blocks, or "microapps";
2. Microapp has its state and provides an API to change that state.
3. Small blocks, called modules handle microapp state and logic.
4. Modules are shape-agnostic, knows little to nothing about an app or other modules.
5. Each module processes one task and nothing more.

## Getting started

A microapp in Stapp terminology is an object, that has only a few fields: `state$`, which is an observable state of an app, and an `api`, which comprises methods to change the state. 

```typescript
type Stapp<State, API> = {
  name: string
  state$: Observable<State>
  api: API
  
  // Imperative and not recommended to use API
  // Still useful for testing
  dispatch: (event: any) => any
  getState: () => State
}
```

Stapp application is a combination of one or more modules.

```typescript
import { createApp } from 'stapp'

const { state$, api } = createApp({
  name: 'My app', // Name is optional
  modules: [
    formBase,
    formHandlers,
    cardValidation({...}),
    cardValidation({...}),
    currencyConvertion({...}),
    swapValues({...}),
    commission({...}),
    payC2C({...})
  ]
})
```

## Modules

CreateApp itself doesn't do much work. It sets up a redux store, a couple of middlewares and returns a Stapp object. Without modules, application api will be empty, and its state will be a plain empty object. Like, forever.

So, what are the modules? A module is a place where all your magic should happen. A module has an ability to:

1. create and handle portions of an app state;
2. provide methods to a public application api;
3. react to state changes;
4. react to api calls.

A basic module is an object or a function, returning an object.

```typescript
type Module<Api, State, FullState = State> = {
  name: string
  dependencies?: string[]

  // Api
  events?: Api
  api?: Api // Alias for events
  waitFor?: Array<AnyEventCreator | string>

  // State
  reducers?: { [K: string]: Reducer<State[K]> }
  state?: { [K: string]: Reducer<State[K]> } // alias for reducers

  // Epics
  epic?: Epic<Partial<Full>>
}
```

### Modules: handling state

Since Stapp uses redux as a state management core, reducers and events are the familiar redux concepts.

Reducer is a pure function, that receives an old state and an event, and must return a new state. An event (or an 'action' in redux terminology) is a plain object with only one required field `type`. Stapp follows [flux-standard-action](https://github.com/redux-utilities/flux-standard-action#actions) convention.

Stapp provides some useful tools to simplify creating of reducers and event creators. These tools are
inspired by redux-act, but have some tweaks and API differences to make the process even simpler.

Here is an example:

```typescript
import { createEvent, createReducer } from 'stapp'

const setValue = createEvent('Set value')
const clearValue = createEvent('Clear value')
const setError = createEvent('Set errror')
const reset = createEvent('Reset state')

const valuesReducer = createReducer({})
  .on(setValue, (values, newValues) => ({ ...values, ...newValues }))
  .on(clearValue, (values, fieldName) => ({ ...values, [fieldName]: null }))
  .reset(reset)

const errorsReducer = createReducer({})
  .on(setError, (errors, newErrors) => ({ ...errros, ...newErrors }))
  .reset(reset)

const formBase = {
  name: FORM_BASE,
  state: { // or `reducers`
    values: valuesReducer,
    errors: errorsReducer
  }
}
```

A reducer created by createReducer is a function with some additional methods. Still, these reducers are just functions, so they can be combined into one with redux `combineReducers` method. And remember, you are not forced to use createReducer at all.

Value of `state` (or `reducers`) fields of every module in the app will be merged into one object and combined into one root reducer, constructing the whole state of an app.

### Modules: events

There are several ways to dispatch events to the store - each for its case.

> If you are familiar with redux, you might ask: — Why do we call actions "events"? See, Flux and, specifically, redux borrowed much from CQRS and Event Sourcing patterns. Using the term "event" instead of "action" prompts to treat redux actions not only as commands to execute but also as events. Still, it's just an opinion, and you may use any tools compatible with flux-standard-action.

#### User interactions

If a module needs to handle some user interactions, it can add methods to the application's public API.
Any event creators passed through an `api` field, will be bound to the store and exposed to an application consumer as API methods.

```typescript
import { createEvent } from 'stapp'

export const formBase = {
  name: FORM_BASE,
  api: { // or `events`
    setValue: createEvent('Set new values')
  }
}
```

```html
// somewhere later
<input name='example' onchange='event => api.setValue({
  example: event.target.value
})' />
```

#### Modules logic

All other events needed for business logic should not be exposed to the public API. Instead, they should be dispatched by so-called epics. The concept of epics will be explained later.

#### Testing

The application object created with `createApp` has `dispatch` and `getState` methods. Although these methods can be used anywhere, the only reason to have them is that they are beneficial in unit tests. So please don't use them anywhere except tests.

### Modules: epics

Epic is one the core concepts, which makes your application reactive. Epic can *react* to anything that happens with the application, and it should be the only place containing business-logic.

Epic is a function, that receives a stream of events and a stream of state, and must return a stream of events. The concept of epics is well explained [here](https://redux-observable.js.org/docs/basics/Epics.html).

The only difference between redux-observable epics and Stapp epics is that the latter accepts a stream of a state as the last argument.

Here is an example of an epic:

```typescript
import { map } from 'rxjs/operators/map'
import { select, combineEpics, createEvent } from 'stapp'
import { setValue } from 'stapp/lib/modules/formBase'

const handleChange = createEvent('handle input change', event => ({
  [event.target.name]: event.target.value
}))

const handleChangeEpic = (event$) => select(handleChange, event$).pipe(
  map(({ payload }) => setValue(payload))
)

const form = {
  name: 'form',
  api: { handleChange },
  epic: combineEpics([handleChangeEpic])
}
```

Each module can provide only one epic, but you can create as many as you like - and combine them into one with `combineEpics` function.

### Modules: module factories

A module factory is a function that returns a module. You'll find out at some point that most of your modules are module factories. 

Sometimes your modules might have some common dependencies. E.g., a request service. Instead of passing them directly into a module, you should pass dependencies to the `dependencies` field of `createApp` config.

Every function passed to the `modules` field will be called with a value provided to the `dependencies` field.

```typescript
// moduleA.js
import { select } from 'stapp'
import { switchMap, map } from 'rxjs/operators'

import { request } from 'my-services'

const moduleA = ({ request }) => ({
  epic: (event$) => select(someEvent, event$).pipe(
    switchMap(({ payload }) => request('/my-cool-api', payload)),
    map(result => someOtherEvent(result))
  )
})    

// my-app.js
const app = createApp({
  name: 'my app',
  modules: [
    moduleA
  ],
  dependencies: {
    request
  }
})
```

### Modules: higher-order module factories

Another typical pattern related to the 'module factory' concept is creating higher order module factories. They are used to keep modules as independent and autonomous as possible. Let's take a previous piece of code as an example. The `moduleA` is used to get some date obviously. What if at some point we decide to change the request service API? We'll have to rewrite every single module using the request service.

Instead, we could wrap moduleA with some preparing function and provide only needed methods to the module. Here is an example:

```typescript
// moduleA.js
import { select } from 'stapp'
import { switchMap, map } from 'rxjs/operators'

import { request } from 'my-services'

const moduleA = ({ getData }) => ({
  epic: (event$) => select(someEvent, event$).pipe(
    switchMap(({ payload }) => getData(payload)),
    map(result => someOtherEvent(result))
  )
})    

// mapRequest.js
const mapRequest = (mapper, module) => ({ request }) => module(mapper(request))

// my-app.js
const app = createApp({
  name: 'my app',
  modules: [
    mapRequest(
      request => ({
        getData: payload => request('/my-cool-api', payload)
      }),
      moduleA
    )
  ],
  dependencies: {
    request
  }
})
```

## API

Follow the links to see descriptions, definitions, and examples.

### Core

- `createApp`
- `createEvent`
- `createEffect`
- `createReducer`
- `whenReady`

### Epics utilities

- `select`
- `selectArray`
- `combineEpics`

### Core events

- `epicEnd`
- `dangerouslyReplaceState`
- `dangerouslyResetState`

### Modules

#### FormBase

Base form module that handles all form interactions.

- `formBase`
- `FORM_BASE`
- `setValue`
- `setTouched`
- `setError`
- `setActive`
- `setReady`
- `resetForm`
- `submit`
- `isValidSelector`
- `isReadySelector`
- `isDirtySelector`
- `isPristineSelector`
- `fieldSelector`

#### Persist

- `persist`

### React utilities

- `createConsumer`
- `createConsume`
- `createForm`
- `createField`

## Peer dependencies

Stapp is dependent on redux, fbjs and RxJS. React-bindings are dependent on react, reselect and prop-types.

The full peer-dependencies list looks like this:

```json
  {
    "fbjs": ">=0.8",
    "prop-types": ">=15.6",
    "react": ">=15",
    "redux": ">=3 <4",
    "rxjs": ">=5",
    "reselect": ">=3"
  }
```

## License

```
Copyright 2018 Tinkoff Bank

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```



## Index

### External modules

* ["async/createAsyncMiddleware/createAsyncMiddleware"](modules/_async_createasyncmiddleware_createasyncmiddleware_.md)
* ["async/createAsyncMiddleware/createAsyncMiddleware.h"](modules/_async_createasyncmiddleware_createasyncmiddleware_h_.md)
* ["core/createApp/createApp"](modules/_core_createapp_createapp_.md)
* ["core/createApp/createApp.h"](modules/_core_createapp_createapp_h_.md)
* ["core/createEffect/createEffect"](modules/_core_createeffect_createeffect_.md)
* ["core/createEffect/createEffect.h"](modules/_core_createeffect_createeffect_h_.md)
* ["core/createEvent/createEvent"](modules/_core_createevent_createevent_.md)
* ["core/createEvent/createEvent.h"](modules/_core_createevent_createevent_h_.md)
* ["core/createReducer/createReducer"](modules/_core_createreducer_createreducer_.md)
* ["core/createReducer/createReducer.h"](modules/_core_createreducer_createreducer_h_.md)
* ["epics/combineEpics/combineEpics"](modules/_epics_combineepics_combineepics_.md)
* ["epics/createStateStreamEnhancer/createStateStreamEnhancer"](modules/_epics_createstatestreamenhancer_createstatestreamenhancer_.md)
* ["epics/select/select"](modules/_epics_select_select_.md)
* ["events/dangerous"](modules/_events_dangerous_.md)
* ["events/epicEnd"](modules/_events_epicend_.md)
* ["events/initDone"](modules/_events_initdone_.md)
* ["helpers/awaitStore/awaitStore"](modules/_helpers_awaitstore_awaitstore_.md)
* ["helpers/constants"](modules/_helpers_constants_.md)
* ["helpers/controlledPromise/controlledPromise"](modules/_helpers_controlledpromise_controlledpromise_.md)
* ["helpers/diffSet/diffSet"](modules/_helpers_diffset_diffset_.md)
* ["helpers/falsify/falsify"](modules/_helpers_falsify_falsify_.md)
* ["helpers/getEpics/getEpics"](modules/_helpers_getepics_getepics_.md)
* ["helpers/getEventType/getEventType"](modules/_helpers_geteventtype_geteventtype_.md)
* ["helpers/getEvents/getEvents"](modules/_helpers_getevents_getevents_.md)
* ["helpers/getModules/getModules"](modules/_helpers_getmodules_getmodules_.md)
* ["helpers/getReducer/getReducer"](modules/_helpers_getreducer_getreducer_.md)
* ["helpers/has/has"](modules/_helpers_has_has_.md)
* ["helpers/identity/identity"](modules/_helpers_identity_identity_.md)
* ["helpers/isArray/isArray"](modules/_helpers_isarray_isarray_.md)
* ["helpers/isEvent/isEvent"](modules/_helpers_isevent_isevent_.md)
* ["helpers/merge/merge"](modules/_helpers_merge_merge_.md)
* ["helpers/t/t"](modules/_helpers_t_t_.md)
* ["helpers/testHelpers/collectEvents/collectEvents"](modules/_helpers_testhelpers_collectevents_collectevents_.md)
* ["helpers/testHelpers/getInitialState/getInitialState"](modules/_helpers_testhelpers_getinitialstate_getinitialstate_.md)
* ["helpers/testHelpers/loggerModule/loggerModule"](modules/_helpers_testhelpers_loggermodule_loggermodule_.md)
* ["helpers/uniqueId/uniqueId"](modules/_helpers_uniqueid_uniqueid_.md)
* ["models/helpers.h"](modules/_models_helpers_h_.md)
* ["modules/formBase/constants"](modules/_modules_formbase_constants_.md)
* ["modules/formBase/events"](modules/_modules_formbase_events_.md)
* ["modules/formBase/formBase"](modules/_modules_formbase_formbase_.md)
* ["modules/formBase/formBase.h"](modules/_modules_formbase_formbase_h_.md)
* ["modules/formBase/reducers"](modules/_modules_formbase_reducers_.md)
* ["modules/formBase/selectors"](modules/_modules_formbase_selectors_.md)
* ["modules/persist/persist"](modules/_modules_persist_persist_.md)
* ["modules/persist/persist.h"](modules/_modules_persist_persist_h_.md)
* ["react/createConsume/createConsume"](modules/_react_createconsume_createconsume_.md)
* ["react/createConsume/createConsume.h"](modules/_react_createconsume_createconsume_h_.md)
* ["react/createConsumer/createConsumer"](modules/_react_createconsumer_createconsumer_.md)
* ["react/createConsumer/createConsumer.h"](modules/_react_createconsumer_createconsumer_h_.md)
* ["react/createField/createField"](modules/_react_createfield_createfield_.md)
* ["react/createField/createField.h"](modules/_react_createfield_createfield_h_.md)
* ["react/createForm/createForm"](modules/_react_createform_createform_.md)
* ["react/createForm/createForm.h"](modules/_react_createform_createform_h_.md)
* ["react/helpers/getDisplayName"](modules/_react_helpers_getdisplayname_.md)
* ["react/helpers/propTypes"](modules/_react_helpers_proptypes_.md)
* ["react/helpers/renderComponent"](modules/_react_helpers_rendercomponent_.md)



---
