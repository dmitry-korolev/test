
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

### Interfaces

* [ConsumerHoc](interfaces/consumerhoc.md)


### Type aliases

* [AnyEventCreator](#anyeventcreator)
* [AnyModule](#anymodule)
* [AnyPayloadTransformer](#anypayloadtransformer)
* [BaseEventCreator](#baseeventcreator)
* [CompleteMeta](#completemeta)
* [ConsumerApi](#consumerapi)
* [ConsumerProps](#consumerprops)
* [Diff](#diff)
* [Effect](#effect)
* [EffectFn](#effectfn)
* [EmptyEventCreator](#emptyeventcreator)
* [Epic](#epic)
* [Event](#event)
* [EventCreator0](#eventcreator0)
* [EventCreator1](#eventcreator1)
* [EventCreator2](#eventcreator2)
* [EventCreator3](#eventcreator3)
* [EventCreators](#eventcreators)
* [EventEpic](#eventepic)
* [EventHandler](#eventhandler)
* [EventHandlers](#eventhandlers)
* [FieldApi](#fieldapi)
* [FieldProps](#fieldprops)
* [FormApi](#formapi)
* [FormBaseState](#formbasestate)
* [FormProps](#formprops)
* [Module](#module)
* [ModuleFactory](#modulefactory)
* [Omit](#omit)
* [PayloadTransformer0](#payloadtransformer0)
* [PayloadTransformer1](#payloadtransformer1)
* [PayloadTransformer2](#payloadtransformer2)
* [PayloadTransformer3](#payloadtransformer3)
* [PersistConfig](#persistconfig)
* [PersistStorage](#persiststorage)
* [Reducer](#reducer)
* [RenderProps](#renderprops)
* [Stapp](#stapp)


### Variables

* [APP_KEY](#app_key)
* [CHAIN_ID](#chain_id)
* [COMPLETE](#complete)
* [FORM_BASE](#form_base)
* [SOURCE_MODULE](#source_module)
* [clearStorage](#clearstorage)
* [consumers](#consumers)
* [dangerouslyReplaceState](#dangerouslyreplacestate)
* [dangerouslyReplaceStateType](#dangerouslyreplacestatetype)
* [dangerouslyResetState](#dangerouslyresetstate)
* [dangerouslyResetStateType](#dangerouslyresetstatetype)
* [epicEnd](#epicend)
* [i](#i)
* [initDone](#initdone)
* [names](#names)
* [promises](#promises)
* [renderPropType](#renderproptype)
* [resetForm](#resetform)
* [selectorType](#selectortype)
* [setActive](#setactive)
* [setError](#seterror)
* [setReady](#setready)
* [setTouched](#settouched)
* [setValue](#setvalue)
* [submit](#submit)


### Functions

* [T](#t)
* [awaitStore](#awaitstore)
* [checkDuplications](#checkduplications)
* [collectEvents](#collectevents)
* [combineEpics](#combineepics)
* [commonHandler](#commonhandler)
* [controlledPromise](#controlledpromise)
* [create](#create)
* [createApp](#createapp)
* [createAsyncMiddleware](#createasyncmiddleware)
* [createConsume](#createconsume)
* [createConsumer](#createconsumer)
* [createEffect](#createeffect)
* [createEvent](#createevent)
* [createField](#createfield)
* [createForm](#createform)
* [createFormBaseReducers](#createformbasereducers)
* [createReducer](#createreducer)
* [createStateStreamEnhancer](#createstatestreamenhancer)
* [defaultMergeProps](#defaultmergeprops)
* [diffSet](#diffset)
* [falsify](#falsify)
* [fieldSelector](#fieldselector)
* [formBase](#formbase)
* [getDisplayName](#getdisplayname)
* [getEpics](#getepics)
* [getEventType](#geteventtype)
* [getEvents](#getevents)
* [getInitialState](#getinitialstate)
* [getModules](#getmodules)
* [getPersistEpics](#getpersistepics)
* [getReducer](#getreducer)
* [has](#has)
* [identity](#identity)
* [isArray](#isarray)
* [isDirtySelector](#isdirtyselector)
* [isError](#iserror)
* [isEvent](#isevent)
* [isPristineSelector](#ispristineselector)
* [isReadySelector](#isreadyselector)
* [isValidSelector](#isvalidselector)
* [mapModule](#mapmodule)
* [mapObject](#mapobject)
* [merge](#merge)
* [persist](#persist)
* [renderComponent](#rendercomponent)
* [run](#run)
* [select](#select)
* [selectArray](#selectarray)
* [uniqueId](#uniqueid)
* [useReduxEnhancer](#usereduxenhancer)
* [whenReady](#whenready)


### Object literals

* [consumerPropTypes](#consumerproptypes)
* [loggerModule](#loggermodule)



---
# Type aliases
<a id="anyeventcreator"></a>

### «Private» AnyEventCreator

**Τ AnyEventCreator**:  *[EmptyEventCreator](#emptyeventcreator)⎮[EventCreator1](#eventcreator1)`any`, `Payload`, `Meta`⎮[EventCreator2](#eventcreator2)`any`, `any`, `Payload`, `Meta`⎮[EventCreator3](#eventcreator3)`any`, `any`, `any`, `Payload`, `Meta`* 

*Defined in core/createEvent/createEvent.h.ts:90*






___

<a id="anymodule"></a>

### «Private» AnyModule

**Τ AnyModule**:  *[ModuleFactory](#modulefactory)`Extra`, `Api`, `State`, `Full`⎮[Module](#module)`Api`, `State`, `Full`* 

*Defined in core/createApp/createApp.h.ts:204*






___

<a id="anypayloadtransformer"></a>

### «Private» AnyPayloadTransformer

**Τ AnyPayloadTransformer**:  *[PayloadTransformer0](#payloadtransformer0)`any`⎮[PayloadTransformer1](#payloadtransformer1)`any`, `any`⎮[PayloadTransformer2](#payloadtransformer2)`any`, `any`, `any`⎮[PayloadTransformer3](#payloadtransformer3)`any`, `any`, `any`, `any`* 

*Defined in core/createEvent/createEvent.h.ts:99*






___

<a id="baseeventcreator"></a>

### «Private» BaseEventCreator

**Τ BaseEventCreator**:  *`object`* 

*Defined in core/createEvent/createEvent.h.ts:26*



#### Type declaration



 epic : function
► **epic**State(fn: *[EventEpic](#eventepic)`Payload`, `Meta`, `State`*): [Epic](#epic)`State`



*Defined in core/createEvent/createEvent.h.ts:29*



**Type parameters:**

#### State 
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| fn | [EventEpic](#eventepic)`Payload`, `Meta`, `State`   |  - |





**Returns:** [Epic](#epic)`State`





 getType : function
► **getType**(): `string`



*Defined in core/createEvent/createEvent.h.ts:27*





**Returns:** `string`





 is : function
► **is**(event: *[Event](#event)`any`, `any`*): `boolean`



*Defined in core/createEvent/createEvent.h.ts:28*



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| event | [Event](#event)`any`, `any`   |  - |





**Returns:** `boolean`







___

<a id="completemeta"></a>

### «Private» CompleteMeta

**Τ CompleteMeta**:  *`object`* 

*Defined in async/createAsyncMiddleware/createAsyncMiddleware.h.ts:6*



#### Type declaration





___

<a id="consumerapi"></a>

###  ConsumerApi

**Τ ConsumerApi**:  *`object`* 

*Defined in react/createConsumer/createConsumer.h.ts:9*


#### Type declaration




 api: `Api`






 state: `State`







___

<a id="consumerprops"></a>

### «Private» ConsumerProps

**Τ ConsumerProps**:  *`object`[RenderProps](#renderprops)[ConsumerApi](#consumerapi)`any`, `any`* 

*Defined in react/createConsumer/createConsumer.h.ts:21*



Type inferring will be available... someday Typings are still in... progress




___

<a id="diff"></a>

### «Private» Diff

**Τ Diff**:  *`({ [P in T]: P; } &amp; { [P in U]: never; } &amp; { [x: string]: never; })[T]`* 

*Defined in models/helpers.h.ts:4*






___

<a id="effect"></a>

###  Effect

**Τ Effect**:  *`object`* 

*Defined in core/createEffect/createEffect.h.ts:4*


#### Type declaration
►(payload: *`Payload`*): `Observable`.<[Event](#event)`any`, `any`>



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| payload | `Payload`   |  - |





**Returns:** `Observable`.<[Event](#event)`any`, `any`>





 fail: [EventCreator1](#eventcreator1)`any`






 start: [EventCreator1](#eventcreator1)`Payload`






 success: [EventCreator1](#eventcreator1)`Result`





 getType : function
► **getType**(): `string`



*Defined in core/createEffect/createEffect.h.ts:10*





**Returns:** `string`





 use : function
► **use**(effectFn: *[EffectFn](#effectfn)`Payload`, `Result`*): [Effect](#effect)`Payload`, `Result`



*Defined in core/createEffect/createEffect.h.ts:9*



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| effectFn | [EffectFn](#effectfn)`Payload`, `Result`   |  - |





**Returns:** [Effect](#effect)`Payload`, `Result`







___

<a id="effectfn"></a>

###  EffectFn

**Τ EffectFn**:  *`function`* 

*Defined in core/createEffect/createEffect.h.ts:30*



EffectCreator function, should return promise or a value

### Example

    const fetchUser = createEffect('Fetch user')
    const effectFunction = // EffectFn<number, UserData>
       (userId: number) => fetch(API_URL + `/users/${userUd}/`)
         .then(result => result.json())
    
    fetchUser.use(effectFunction)
    store.dispatch(fetchUser(10))

#### Type declaration
►(payload: *`Payload`*): `Promise`.<`Result`>⎮`Result`



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| payload | `Payload`   |  - |





**Returns:** `Promise`.<`Result`>⎮`Result`






___

<a id="emptyeventcreator"></a>

### «Private» EmptyEventCreator

**Τ EmptyEventCreator**:  *[BaseEventCreator](#baseeventcreator)`void`, `void``function`* 

*Defined in core/createEvent/createEvent.h.ts:35*






___

<a id="epic"></a>

###  Epic

**Τ Epic**:  *[EventEpic](#eventepic)`any`, `any`, `State`* 

*Defined in core/createApp/createApp.h.ts:47*



Epic is a function, that must return an Observable. An observable may emit any value, but only valid events will be passed to dispatch.

### Example

     import { select } from 'stapp'
     import { setValues } from 'stapp/lib/modules/formBase'
    
     const handleChange = createEvent('Handle change')
     const handleChangeEpic = (action$, state$) => select(handleChange, action$).pipe(
       map(event => setValues({ [event.target.name]: event.target.value })
     )
*__param__*: Stream of events

*__param__*: Stream of state





___

<a id="event"></a>

###  Event

**Τ Event**:  *`object`* 

*Defined in core/createEvent/createEvent.h.ts:16*


#### Type declaration




 error: `boolean`






 meta: `Meta`






 payload: `Payload`






 type: `string`







___

<a id="eventcreator0"></a>

### «Private» EventCreator0

**Τ EventCreator0**:  *[BaseEventCreator](#baseeventcreator)`Payload`, `Meta``function`* 

*Defined in core/createEvent/createEvent.h.ts:40*






___

<a id="eventcreator1"></a>

### «Private» EventCreator1

**Τ EventCreator1**:  *[BaseEventCreator](#baseeventcreator)`Payload`, `Meta``function`* 

*Defined in core/createEvent/createEvent.h.ts:46*






___

<a id="eventcreator2"></a>

### «Private» EventCreator2

**Τ EventCreator2**:  *[BaseEventCreator](#baseeventcreator)`Payload`, `Meta``function`* 

*Defined in core/createEvent/createEvent.h.ts:52*






___

<a id="eventcreator3"></a>

### «Private» EventCreator3

**Τ EventCreator3**:  *[BaseEventCreator](#baseeventcreator)`Payload`, `Meta``function`* 

*Defined in core/createEvent/createEvent.h.ts:58*






___

<a id="eventcreators"></a>

###  EventCreators

**Τ EventCreators**:  *`object`* 

*Defined in core/createEvent/createEvent.h.ts:108*



An object with various event creators as values

#### Type declaration


[K: `string`]: [AnyEventCreator](#anyeventcreator)






___

<a id="eventepic"></a>

###  EventEpic

**Τ EventEpic**:  *`function`* 

*Defined in core/createApp/createApp.h.ts:23*



### Example

`` ` ``typescript import { setValues } from 'stapp/lib/modules/formBase'

const handleChange = createEvent('Handle change') const handleChangeEpic = handleChange.epic((handleChange$, state$) => handleChange$.pipe( map(event => setValues({ \[event.target.name\]: event.target.value }) )) `` ` ``
*__param__*: Stream of events

*__param__*: Stream of state


#### Type declaration
►(event$: *`Observable`.<[Event](#event)`Payload`, `Meta`>*, state$: *`Observable`.<`State`>*): `Observable`.<`any`>



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| event$ | `Observable`.<[Event](#event)`Payload`, `Meta`>   |  - |
| state$ | `Observable`.<`State`>   |  - |





**Returns:** `Observable`.<`any`>






___

<a id="eventhandler"></a>

###  EventHandler

**Τ EventHandler**:  *`function`* 

*Defined in core/createReducer/createReducer.h.ts:20*



Event handler. Accepts current state, event payload and event meta. Should return new state.

### Example

     import { createEvent, createReducer } from 'stapp'
    
     const add = createEvent<number>('Add value')
     const addHandler = (state, payload) => state + payload
    
     const reducer = createReducer(0).on(add, addHandler)

#### Type declaration
►(state: *`State`*, payload: *`Payload`*, meta: *`Meta`*): `State`



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| state | `State`   |  - |
| payload | `Payload`   |  - |
| meta | `Meta`   |  - |





**Returns:** `State`






___

<a id="eventhandlers"></a>

###  EventHandlers

**Τ EventHandlers**:  *`object`* 

*Defined in core/createReducer/createReducer.h.ts:138*



An object with various event handlers as values

#### Type declaration





___

<a id="fieldapi"></a>

###  FieldApi

**Τ FieldApi**:  *`object`* 

*Defined in react/createField/createField.h.ts:4*


#### Type declaration




 input: `object`








 name: `string`






 onBlur: `function`




►(event: *`SyntheticEvent`.<`any`>*): `void`



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| event | `SyntheticEvent`.<`any`>   |  - |





**Returns:** `void`






 onChange: `function`




►(event: *`SyntheticEvent`.<`any`>*): `void`



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| event | `SyntheticEvent`.<`any`>   |  - |





**Returns:** `void`






 onFocus: `function`




►(event: *`SyntheticEvent`.<`any`>*): `void`



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| event | `SyntheticEvent`.<`any`>   |  - |





**Returns:** `void`






 value: `string`







 meta: `object`








 active: `boolean`






 error: `any`






 touched: `boolean`








___

<a id="fieldprops"></a>

###  FieldProps

**Τ FieldProps**:  *[RenderProps](#renderprops)[FieldApi](#fieldapi)`object`* 

*Defined in react/createField/createField.h.ts:19*





___

<a id="formapi"></a>

###  FormApi

**Τ FormApi**:  *`object`* 

*Defined in react/createForm/createForm.h.ts:4*


#### Type declaration




 dirty: `boolean`






 handleSubmit: `function`




►(event: *`SyntheticEvent`.<`any`>*): `void`



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| event | `SyntheticEvent`.<`any`>   |  - |





**Returns:** `void`






 pristine: `boolean`






 ready: `boolean`






 valid: `boolean`







___

<a id="formbasestate"></a>

###  FormBaseState

**Τ FormBaseState**:  *`object`* 

*Defined in modules/formBase/formBase.h.ts:1*


#### Type declaration




 active: `keyof Values`⎮`null`






 dirty: `object`









 errors: `object`









 pristine: `boolean`






 ready: `object`









 touched: `object`









 values: `Values`







___

<a id="formprops"></a>

###  FormProps

**Τ FormProps**:  *[RenderProps](#renderprops)[FormApi](#formapi)* 

*Defined in react/createForm/createForm.h.ts:12*





___

<a id="module"></a>

###  Module

**Τ Module**:  *`object`* 

*Defined in core/createApp/createApp.h.ts:73*



Module is one two core concepts of Stapp. Basically, a module is a plain object that consists of several fields. The only required field is `name`. It will be used in development mode for debugging.

### Example

     // Typical module may look like this
     const formHandlers = {
       name: 'formHandlers',
       dependencies: ['formBase'],
       events: {
         handleChange: createEvent(...)
       },
       epic: combineEpics([...])
     }
    

See more examples at README.md
*__see__*: [ModuleFactory](#modulefactory)


#### Type declaration




«Optional»  api: [Api]()






«Optional»  dependencies: `string`[]






«Optional»  epic: [Epic](#epic)`Partial`.<`Full`>






«Optional»  events: [Api]()






 name: `string`






«Optional»  reducers: `undefined`⎮`object`






«Optional»  state: `undefined`⎮`object`






«Optional»  waitFor: `Array`.<[AnyEventCreator](#anyeventcreator)⎮`string`>







___

<a id="modulefactory"></a>

###  ModuleFactory

**Τ ModuleFactory**:  *`function`* 

*Defined in core/createApp/createApp.h.ts:190*



ModuleFactory is a function that must return a [Module](#module). It will be called with any extra dependencies passed to [createApp](#createapp).

### Example

     import { submit } from 'stapp/lib/modules/formBase'
    
     const paymentModule = ({ pay }) => {
       const payEffect = createEffect('Perform payment', pay)
    
       const payEpic = (event$, state$) => state$.pipe(
         sample(select(submit, event$)),
         switchMap(state => payEffect(state.values))
       )
    
       const paymentInfo = createReducer({})
         .on(payEffect.success, (_, payload) => payload)
         .on(payEffect.fail, (_, error) => error)
    
       return {
         name: 'payModule',
         reducers: {
           paymentInfo
         },
         epic: payEpic
       }
     }
    
     // later
     const app = createApp({
       name: 'Pay form',
       modules: [
         formBase,
         paymentModule
       ],
       dependencies: {
         pay (values) {...}
       }
     })
    

### Higher order module factory

One of the common patterns is using higher order module factories. E.g. Stapp application is being used within another app, based on class react-redux stack, and one of Stapp modules need some data from global redux store. It would be nice, if module could get only what it needed without knowing about the source of the data.

    
     import { from } from 'rxjs/observable/from'
     import { map } from 'rxjs/operators/map'
    
     const someModule = ({ state$ }) => ({
       name: 'someModule',
       epic: (action$) => {
         // some logic dependent on `state$` data
       }
     })
    
     // That's just an example!
     const connect = (config, moduleFactory) => ({ context }) => {
       const store = config[config.storeKey] || context[config.storeKey]
    
       // Note that module factories are called only once, so we can't provide state itself,
       // instead module needs some sort of subscribable source. E.g. observable.
       const state$ = from(store).pipe(map(config.mapState))
    
       return moduleFactory({ state$ })
     }
    
     // later is a component
     static contextTypes = {
       store: pt.object.isRequired
     }
    
     componentWillMount () {
       this.app = createApp({
         name: 'connected app',
         modules: [
           connect({
             mapState: state => state.providers,
             storeKey: 'store'
           }, someModule)
         ],
         dependencies: {
           context: this.context
         }
       })
     }
    

As a result we have a module, that knows nothing about global state and receives only what it needs and a higher order module, that connects other module factories to global store.

See more examples at README.md

#### Type declaration
►(extraArgument: *`Extra`*): [Module](#module)`Api`, `State`, `Full`



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| extraArgument | `Extra`   |  - |





**Returns:** [Module](#module)`Api`, `State`, `Full`






___

<a id="omit"></a>

### «Private» Omit

**Τ Omit**:  *`object`* 

*Defined in models/helpers.h.ts:10*



#### Type declaration





___

<a id="payloadtransformer0"></a>

###  PayloadTransformer0

**Τ PayloadTransformer0**:  *`function`* 

*Defined in core/createEvent/createEvent.h.ts:64*



#### Type declaration
►(): `Payload`





**Returns:** `Payload`






___

<a id="payloadtransformer1"></a>

###  PayloadTransformer1

**Τ PayloadTransformer1**:  *`function`* 

*Defined in core/createEvent/createEvent.h.ts:70*



#### Type declaration
►(arg1: *`A1`*): `Payload`



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| arg1 | `A1`   |  - |





**Returns:** `Payload`






___

<a id="payloadtransformer2"></a>

###  PayloadTransformer2

**Τ PayloadTransformer2**:  *`function`* 

*Defined in core/createEvent/createEvent.h.ts:77*



#### Type declaration
►(arg1: *`A1`*, arg2: *`A2`*): `Payload`



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| arg1 | `A1`   |  - |
| arg2 | `A2`   |  - |





**Returns:** `Payload`






___

<a id="payloadtransformer3"></a>

###  PayloadTransformer3

**Τ PayloadTransformer3**:  *`function`* 

*Defined in core/createEvent/createEvent.h.ts:85*



#### Type declaration
►(arg1: *`A1`*, arg2: *`A2`*, arg3: *`A3`*): `Payload`



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| arg1 | `A1`   |  - |
| arg2 | `A2`   |  - |
| arg3 | `A3`   |  - |





**Returns:** `Payload`






___

<a id="persistconfig"></a>

###  PersistConfig

**Τ PersistConfig**:  *`object`* 

*Defined in modules/persist/persist.h.ts:11*



Persist module config

#### Type declaration




«Optional»  debounce: `undefined`⎮`number`






«Optional»  getter: `undefined`⎮`function`






 key: `string`






«Optional»  setter: `undefined`⎮`function`






 storage: [PersistStorage](#persiststorage)







___

<a id="persiststorage"></a>

###  PersistStorage

**Τ PersistStorage**:  *`object`* 

*Defined in modules/persist/persist.h.ts:1*


#### Type declaration



 getItem : function
► **getItem**(key: *`string`*): `string`⎮`null`⎮`undefined`



*Defined in modules/persist/persist.h.ts:2*



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| key | `string`   |  - |





**Returns:** `string`⎮`null`⎮`undefined`





 removeItem : function
► **removeItem**(key: *`string`*): `void`



*Defined in modules/persist/persist.h.ts:3*



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| key | `string`   |  - |





**Returns:** `void`





 setItem : function
► **setItem**(key: *`string`*, data: *`string`*): `void`



*Defined in modules/persist/persist.h.ts:4*



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| key | `string`   |  - |
| data | `string`   |  - |





**Returns:** `void`







___

<a id="reducer"></a>

###  Reducer

**Τ Reducer**:  *`object`* 

*Defined in core/createReducer/createReducer.h.ts:31*



Basically, a reducer is a function, that accepts a state and an event, and returns new state. Stapp [createReducer](#createreducer) creates a reducer on steroids. See examples below.

#### Type declaration
►(state: *`State`*, event: *[Event](#event)`any`, `any`*): `State`




**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| state | `State`   |  Reducer's state |
| event | [Event](#event)`any`, `any`   |  Any event |





**Returns:** `State`
new state





 createEvents : function
► **createEvents**T(model: *`object`*): `object`



*Defined in core/createReducer/createReducer.h.ts:128*



Creates an object of eventCreators from passed eventHandlers.

### Example

     const { add, subtract, double } = reducer.createEvents({
       add: (state, payload) => state + payload,
       subtract: (state, payload) => state - payload,
       double: (state) => state * 2
     })


**Type parameters:**

#### T :  `string`

List of eventCreators names

**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| model | `object`   |  Object of event handlers |





**Returns:** `object`





 has : function
► **has**(event: *[AnyEventCreator](#anyeventcreator)⎮`string`*): `boolean`



*Defined in core/createReducer/createReducer.h.ts:111*



Checks if the reducer has a handler for provided event creator or event type.

### Example

     import { userLogout } from '../../someModule'
     reducer.has(userLogout) // true


**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| event | [AnyEventCreator](#anyeventcreator)⎮`string`   |  Any event creator or a string representing event type |





**Returns:** `boolean`





 off : function
► **off**(event: *[AnyEventCreator](#anyeventcreator)⎮`string`⎮`Array`.<[AnyEventCreator](#anyeventcreator)⎮`string`>*): [Reducer](#reducer)`State`



*Defined in core/createReducer/createReducer.h.ts:84*



Detaches EventHandler from the reducer. Returns reducer.

### Example

     reducer.on(double, (state) => {
       if (state > 1000) {
         reducer.off(double)
       }
       return state * 2
     })


**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| event | [AnyEventCreator](#anyeventcreator)⎮`string`⎮`Array`.<[AnyEventCreator](#anyeventcreator)⎮`string`>   |  Event creator, a string representing event type or an array of event creators or types |





**Returns:** [Reducer](#reducer)`State`
Same reducer






 on : function
► **on**Payload,Meta(event: *[AnyEventCreator](#anyeventcreator)`Payload`, `Meta`⎮`string`⎮`Array`.<[AnyEventCreator](#anyeventcreator)⎮`string`>*, handler: *[EventHandler](#eventhandler)`State`, `Payload`, `Meta`*): [Reducer](#reducer)`State`



*Defined in core/createReducer/createReducer.h.ts:63*



Attaches EventHandler to the reducer. Adding handlers of already existing event type will override previous handlers. Returns reducer.

### Example

     import { createEvent, createReducer } from 'stapp'
    
     const double = createEvent('Double state)
     const add = createEvent<number>('Add value')
     const anotherAdd = createEvent('Add string', (x: string) => +x)
    
     const reducer = createReducer(0)
       .on(double, (state) => state * 2) // Single event creator
       .on([add, anotherAdd], (state, payload) => state + payload) // An array of event creators
       .on('SUBTRACT', (state, payload) => state - payload) // Or a string representing event type


**Type parameters:**

#### Payload 

Event payload

#### Meta 

Event meta data

**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| event | [AnyEventCreator](#anyeventcreator)`Payload`, `Meta`⎮`string`⎮`Array`.<[AnyEventCreator](#anyeventcreator)⎮`string`>   |  Event creator, a string representing event type or an array of event creators or types |
| handler | [EventHandler](#eventhandler)`State`, `Payload`, `Meta`   |  EventHandler |





**Returns:** [Reducer](#reducer)`State`
same reducer






 reset : function
► **reset**(event: *[AnyEventCreator](#anyeventcreator)⎮`string`⎮`Array`.<[AnyEventCreator](#anyeventcreator)⎮`string`>*): [Reducer](#reducer)`State`



*Defined in core/createReducer/createReducer.h.ts:98*



Assigns reset handler to provided event types. Reset handler always returns initialState.

### Example

     import { userLogout } from '../../someModule'
     reducer.reset(userLogout)


**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| event | [AnyEventCreator](#anyeventcreator)⎮`string`⎮`Array`.<[AnyEventCreator](#anyeventcreator)⎮`string`>   |  Event creator, a string representing event type or an array of event creators or types |





**Returns:** [Reducer](#reducer)`State`
Same reducer








___

<a id="renderprops"></a>

###  RenderProps

**Τ RenderProps**:  *`object`* 

*Defined in react/createConsumer/createConsumer.h.ts:3*


#### Type declaration




«Optional»  children: `undefined`⎮`function`






«Optional»  component: `ReactType`






«Optional»  render: `undefined`⎮`function`







___

<a id="stapp"></a>

###  Stapp

**Τ Stapp**:  *`object`* 

*Defined in core/createApp/createApp.h.ts:216*



An app, created by [createApp](#createapp) is another core concept of Stapp. See README.md for details.

#### Type declaration




 api: `Api`






 dispatch: `function`




►(event: *`any`*): `any`



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| event | `any`   |  - |





**Returns:** `any`






 getState: `function`




►(): `State`





**Returns:** `State`






 name: `string`






 state$: `Observable`.<`State`>







___


# Variables
<a id="app_key"></a>

### «Private»«Const» APP_KEY

**●  APP_KEY**:  *"@@stapp"*  = "@@stapp"

*Defined in helpers/constants.ts:4*






___

<a id="chain_id"></a>

### «Private»«Const» CHAIN_ID

**●  CHAIN_ID**:  *`string`*  =  `${APP_KEY}/chainId`

*Defined in helpers/constants.ts:19*






___

<a id="complete"></a>

### «Private»«Const» COMPLETE

**●  COMPLETE**:  *`string`*  =  `${APP_KEY}/complete`

*Defined in helpers/constants.ts:9*






___

<a id="form_base"></a>

### «Const» FORM_BASE

**●  FORM_BASE**:  *`string`*  =  `${APP_KEY}/formBase`

*Defined in modules/formBase/constants.ts:3*





___

<a id="source_module"></a>

### «Private»«Const» SOURCE_MODULE

**●  SOURCE_MODULE**:  *`string`*  =  `${APP_KEY}/sourceModule`

*Defined in helpers/constants.ts:14*






___

<a id="clearstorage"></a>

### «Private»«Const» clearStorage

**●  clearStorage**:  *[EmptyEventCreator](#emptyeventcreator)*  =  createEvent(`${APP_KEY}/Persist: Clear storage`)

*Defined in modules/persist/persist.ts:28*






___

<a id="consumers"></a>

### «Private»«Const» consumers

**●  consumers**:  *`object`* 

*Defined in react/createConsumer/createConsumer.ts:24*



#### Type declaration


[K: `string`]: `ComponentClass`.<[ConsumerProps]()`any`, `any`, `any`, `any`>






___

<a id="dangerouslyreplacestate"></a>

### «Const» dangerouslyReplaceState

**●  dangerouslyReplaceState**:  *[EventCreator1](#eventcreator1)`any`*  =  createEvent<any>(
  `${APP_KEY}: Replace state`
)

*Defined in events/dangerous.ts:13*



Can be used to replace state completely




___

<a id="dangerouslyreplacestatetype"></a>

### «Private»«Const» dangerouslyReplaceStateType

**●  dangerouslyReplaceStateType**:  *`string`*  =  dangerouslyReplaceState.getType()

*Defined in helpers/getReducer/getReducer.ts:11*






___

<a id="dangerouslyresetstate"></a>

### «Const» dangerouslyResetState

**●  dangerouslyResetState**:  *[EmptyEventCreator](#emptyeventcreator)*  =  createEvent(`${APP_KEY}: Reset state`)

*Defined in events/dangerous.ts:20*



Can be used to reinitialize state




___

<a id="dangerouslyresetstatetype"></a>

### «Private»«Const» dangerouslyResetStateType

**●  dangerouslyResetStateType**:  *`string`*  =  dangerouslyResetState.getType()

*Defined in helpers/getReducer/getReducer.ts:16*






___

<a id="epicend"></a>

### «Const» epicEnd

**●  epicEnd**:  *[EmptyEventCreator](#emptyeventcreator)*  =  createEvent(`${APP_KEY}: Epic enc`)

*Defined in events/epicEnd.ts:10*



Event signaling about epic completing




___

<a id="i"></a>

### «Private»«Let» i

**●  i**:  *`number`*  = 0

*Defined in helpers/uniqueId/uniqueId.ts:4*






___

<a id="initdone"></a>

### «Private»«Const» initDone

**●  initDone**:  *[EmptyEventCreator](#emptyeventcreator)*  =  createEvent(`${APP_KEY}: Initialization complete`)

*Defined in events/initDone.ts:11*



Indicates about completing initialization




___

<a id="names"></a>

### «Private»«Const» names

**●  names**:  *`string`[]*  =  []

*Defined in helpers/awaitStore/awaitStore.ts:9*






___

<a id="promises"></a>

### «Private»«Const» promises

**●  promises**:  *`Array`.<`Promise`.<`any`>>*  =  []

*Defined in helpers/awaitStore/awaitStore.ts:4*






___

<a id="renderproptype"></a>

### «Private»«Const» renderPropType

**●  renderPropType**:  *`Requireable`.<`any`>*  =  pt.func

*Defined in react/helpers/propTypes.ts:6*






___

<a id="resetform"></a>

### «Const» resetForm

**●  resetForm**:  *`object``function`*  =  createEvent(`${FORM_BASE}: Reset form state`)

*Defined in modules/formBase/events.ts:40*



Used to reset form state




___

<a id="selectortype"></a>

### «Private»«Const» selectorType

**●  selectorType**:  *`Requireable`.<`any`>*  =  pt.func

*Defined in react/helpers/propTypes.ts:11*






___

<a id="setactive"></a>

### «Const» setActive

**●  setActive**:  *`object``function`*  =  createEvent<string | null>(`${FORM_BASE}: Set field as active`)

*Defined in modules/formBase/events.ts:30*



Set field as active




___

<a id="seterror"></a>

### «Const» setError

**●  setError**:  *`object``function`*  =  createEvent<{ [K: string]: string }>(`${FORM_BASE}: Set field error`)

*Defined in modules/formBase/events.ts:18*



Used to set errors for fields




___

<a id="setready"></a>

### «Const» setReady

**●  setReady**:  *`object``function`*  =  createEvent<{ [K: string]: boolean }>(`${FORM_BASE}: Set readiness`)

*Defined in modules/formBase/events.ts:35*



Used to set readiness state




___

<a id="settouched"></a>

### «Const» setTouched

**●  setTouched**:  *`object``function`*  =  createEvent<{ [K: string]: boolean }>(
  `${FORM_BASE}: Set field as touched`
)

*Defined in modules/formBase/events.ts:23*



Used to set field as touched




___

<a id="setvalue"></a>

### «Const» setValue

**●  setValue**:  *`object``function`*  =  createEvent<{ [K: string]: any }>(`${FORM_BASE}: Set field value`)

*Defined in modules/formBase/events.ts:13*



Used to set values for fields




___

<a id="submit"></a>

### «Const» submit

**●  submit**:  *`object``function`*  =  createEvent(`${FORM_BASE}: Submit`, () => undefined)

*Defined in modules/formBase/events.ts:45*



Used to indicate form submission




___


# Functions
<a id="t"></a>

### «Private»«Const» T

► **T**(...args: *`any`[]*): `boolean`



*Defined in helpers/t/t.ts:5*



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| args | `any`[]   |  - |





**Returns:** `boolean`







___

<a id="awaitstore"></a>

### «Private»«Const» awaitStore

► **awaitStore**(name: *`string`*, promise: *`Promise`.<`any`>*): `void`



*Defined in helpers/awaitStore/awaitStore.ts:16*



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| name | `string`   |  - |
| promise | `Promise`.<`any`>   |  - |





**Returns:** `void`





___

<a id="checkduplications"></a>

### «Private»«Const» checkDuplications

► **checkDuplications**T(result: *`T`*, fieldData: *`T`*): `void`



*Defined in helpers/merge/merge.ts:10*



Ensures modules not to overwrite properties, set by other modules


**Type parameters:**

#### T 
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| result | `T`   |  - |
| fieldData | `T`   |  - |





**Returns:** `void`





___

<a id="collectevents"></a>

### «Private»«Const» collectEvents

► **collectEvents**T(stream: *`Observable`.<`T`>*): `Promise`.<`T`[]>



*Defined in helpers/testHelpers/collectEvents/collectEvents.ts:7*



**Type parameters:**

#### T 
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| stream | `Observable`.<`T`>   |  - |





**Returns:** `Promise`.<`T`[]>





___

<a id="combineepics"></a>

### «Const» combineEpics

► **combineEpics**S1,S2,S3,S4,S5,S6,S7,S8,S9,S10,State(epics: *[[Epic](#epic)`S1`,[Epic](#epic)`S2`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`,[Epic](#epic)`S5`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`,[Epic](#epic)`S5`,[Epic](#epic)`S6`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`,[Epic](#epic)`S5`,[Epic](#epic)`S6`,[Epic](#epic)`S7`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`,[Epic](#epic)`S5`,[Epic](#epic)`S6`,[Epic](#epic)`S7`,[Epic](#epic)`S8`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`,[Epic](#epic)`S5`,[Epic](#epic)`S6`,[Epic](#epic)`S7`,[Epic](#epic)`S8`,[Epic](#epic)`S9`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`,[Epic](#epic)`S5`,[Epic](#epic)`S6`,[Epic](#epic)`S7`,[Epic](#epic)`S8`,[Epic](#epic)`S9`,[Epic](#epic)`S10`]*): [Epic](#epic)`State`



*Defined in epics/combineEpics/combineEpics.ts:13*



Combines epics into one


**Type parameters:**

#### S1 
#### S2 
#### S3 
#### S4 
#### S5 
#### S6 
#### S7 
#### S8 
#### S9 
#### S10 
#### State 
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| epics | [[Epic](#epic)`S1`,[Epic](#epic)`S2`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`,[Epic](#epic)`S5`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`,[Epic](#epic)`S5`,[Epic](#epic)`S6`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`,[Epic](#epic)`S5`,[Epic](#epic)`S6`,[Epic](#epic)`S7`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`,[Epic](#epic)`S5`,[Epic](#epic)`S6`,[Epic](#epic)`S7`,[Epic](#epic)`S8`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`,[Epic](#epic)`S5`,[Epic](#epic)`S6`,[Epic](#epic)`S7`,[Epic](#epic)`S8`,[Epic](#epic)`S9`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`,[Epic](#epic)`S5`,[Epic](#epic)`S6`,[Epic](#epic)`S7`,[Epic](#epic)`S8`,[Epic](#epic)`S9`,[Epic](#epic)`S10`]   |  An array of epics to combine |





**Returns:** [Epic](#epic)`State`







___

<a id="commonhandler"></a>

### «Private»«Const» commonHandler

► **commonHandler**T,K(o: *`T`*, n: *`Partial`.<`T`>*): `T`



*Defined in modules/formBase/reducers.ts:12*



**Type parameters:**

#### T 
#### K :  `keyof T`
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| o | `T`   |  - |
| n | `Partial`.<`T`>   |  - |





**Returns:** `T`





___

<a id="controlledpromise"></a>

### «Private»«Const» controlledPromise

► **controlledPromise**T(): `object`



*Defined in helpers/controlledPromise/controlledPromise.ts:4*



**Type parameters:**

#### T 


**Returns:** `object`





___

<a id="create"></a>

### «Private»«Const» create

► **create**Payload,Result(description: *`string`*, condition: *`function`*, effect?: *[EffectFn](#effectfn)`Payload`, `Result`*): `any`



*Defined in core/createEffect/createEffect.ts:39*



**Type parameters:**

#### Payload 
#### Result 
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| description | `string`   |  - |
| condition | `function`   |  - |
| effect | [EffectFn](#effectfn)`Payload`, `Result`   |  - |





**Returns:** `any`





___

<a id="createapp"></a>

###  createApp

► **createApp**A1,S1,E1,A2,S2,E2,A3,S3,E3,A4,S4,E4,A5,S5,E5,A6,S6,E6,A7,S7,E7,A8,S8,E8,A9,S9,E9,A10,S10,E10,Extra,State,Api(config: *`object`*): [Stapp](#stapp)`State`, `Api`



*Defined in core/createApp/createApp.ts:37*



Creates an application and returns a [Stapp](#stapp).


**Type parameters:**

#### A1 :  [EventCreators](#eventcreators)
#### S1 
#### E1 
#### A2 :  [EventCreators](#eventcreators)
#### S2 
#### E2 
#### A3 :  [EventCreators](#eventcreators)
#### S3 
#### E3 
#### A4 :  [EventCreators](#eventcreators)
#### S4 
#### E4 
#### A5 :  [EventCreators](#eventcreators)
#### S5 
#### E5 
#### A6 :  [EventCreators](#eventcreators)
#### S6 
#### E6 
#### A7 :  [EventCreators](#eventcreators)
#### S7 
#### E7 
#### A8 :  [EventCreators](#eventcreators)
#### S8 
#### E8 
#### A9 :  [EventCreators](#eventcreators)
#### S9 
#### E9 
#### A10 :  [EventCreators](#eventcreators)
#### S10 
#### E10 
#### Extra :  `E1``E2``E3``E4``E5``E6``E7``E8``E9``E10`
#### State :  `S1``S2``S3``S4``S5``S6``S7``S8``S9``S10`
#### Api :  `A1``A2``A3``A4``A5``A6``A7``A8``A9``A10`
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| config | `object`   |  - |





**Returns:** [Stapp](#stapp)`State`, `Api`







___

<a id="createasyncmiddleware"></a>

### «Private» createAsyncMiddleware

► **createAsyncMiddleware**State(eventCreators?: *`Array`.<[AnyEventCreator](#anyeventcreator)⎮`string`>*): `object`



*Defined in async/createAsyncMiddleware/createAsyncMiddleware.ts:16*




**Type parameters:**

#### State 
**Parameters:**

| Param | Type | Default value | Description |
| ------ | ------ | ------ | ------ |
| eventCreators | `Array`.<[AnyEventCreator](#anyeventcreator)⎮`string`>  |  [] |   - |





**Returns:** `object`





___

<a id="createconsume"></a>

### «Const» createConsume

► **createConsume**State,Api(app: *[Stapp](#stapp)`State`, `Api`*): [ConsumerHoc](interfaces/consumerhoc.md)`State`, `Api`



*Defined in react/createConsume/createConsume.tsx:38*



Creates higher order component, that passes state and api from a Stapp application to a wrapped component

Consume example
---------------

     import { createContext } from 'stapp/lib/react'
     import todoApp from '../myApps/todoApp.js'
     import ListItem from '../components'
    
     const { consume } = createContext(todoApp)
    
     const App = consume()((props) => {
       return prop.todos.map((item) => <ListItem
         { ...item }
         key={ item.id }
         handleClick={ () => prop.handleClick(item.id) }
       />)
     })


**Type parameters:**

#### State 
#### Api 
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| app | [Stapp](#stapp)`State`, `Api`   |  - |





**Returns:** [ConsumerHoc](interfaces/consumerhoc.md)`State`, `Api`





___

<a id="createconsumer"></a>

### «Const» createConsumer

► **createConsumer**State,Api(app: *[Stapp](#stapp)`State`, `Api`*): `ComponentClass`.<[ConsumerProps]()`State`, `Api`, `any`, `any`>



*Defined in react/createConsumer/createConsumer.ts:59*



Creates Consumer component

Consumer example
----------------

     import { createContext } from 'stapp/lib/react'
     import todoApp from '../myApps/todoApp.js'
     import ListItem from '../components'
    
     const { Consumer } = createContext(todoApp)
    
     const App = () => <Consumer>
       {
         ({ state, api }) => state.todos.map((item) => <ListItem
           { ...item }
           key={ item.id }
           handleClick={ () => api.handleClick(item.id) }
         />)
       }
     </Consumer>


**Type parameters:**

#### State 
#### Api 
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| app | [Stapp](#stapp)`State`, `Api`   |  - |





**Returns:** `ComponentClass`.<[ConsumerProps]()`State`, `Api`, `any`, `any`>





___

<a id="createeffect"></a>

### «Const» createEffect

► **createEffect**Payload,Result(description: *`string`*, options?: *[EffectFn](#effectfn)`Payload`, `Result`⎮`object`*): [Effect](#effect)`Payload`, `Result`



*Defined in core/createEffect/createEffect.ts:112*



Creates an effect creator. Effect is a stream, that uses provided [EffectFn](#effectfn), and emits start, success, error and complete types.

### Usage in epic example

     const payEffect = createEffect('Perform payment', pay)
    
     const payEpic = (event$, state$) => state$.pipe(
       sample(select(submit, event$)),
       switchMap(state => payEffect(state.values))
     )
    

### Usage as an api method

    ({ api }) => <Button onClick={api.payEffect} />


**Type parameters:**

#### Payload 
#### Result 
**Parameters:**

| Param | Type | Default value | Description |
| ------ | ------ | ------ | ------ |
| description | `string`  | - |   - |
| options | [EffectFn](#effectfn)`Payload`, `Result`⎮`object`  |  {} |   - |





**Returns:** [Effect](#effect)`Payload`, `Result`







___

<a id="createevent"></a>

###  createEvent

► **createEvent**(description?: *`undefined`⎮`string`*): [EmptyEventCreator](#emptyeventcreator)

► **createEvent**Payload(description?: *`undefined`⎮`string`*): [EventCreator1](#eventcreator1)`Payload`, `Payload`

► **createEvent**Payload,Meta(description: *`string`*, payloadCreator?: *[PayloadTransformer0](#payloadtransformer0)`Payload`*, metaCreator?: *[PayloadTransformer0](#payloadtransformer0)`Meta`*): [EventCreator0](#eventcreator0)`Payload`, `Meta`

► **createEvent**A1,Payload,Meta(description: *`string`*, payloadCreator: *[PayloadTransformer1](#payloadtransformer1)`A1`, `Payload`*, metaCreator?: *[PayloadTransformer1](#payloadtransformer1)`A1`, `Meta`*): [EventCreator1](#eventcreator1)`A1`, `Payload`, `Meta`

► **createEvent**A1,A2,Payload,Meta(description: *`string`*, payloadCreator: *[PayloadTransformer2](#payloadtransformer2)`A1`, `A2`, `Payload`*, metaCreator?: *[PayloadTransformer2](#payloadtransformer2)`A1`, `A2`, `Meta`*): [EventCreator2](#eventcreator2)`A1`, `A2`, `Payload`, `Meta`

► **createEvent**A1,A2,A3,Payload,Meta(description: *`string`*, payloadCreator: *[PayloadTransformer3](#payloadtransformer3)`A1`, `A2`, `A3`, `Payload`*, metaCreator?: *[PayloadTransformer3](#payloadtransformer3)`A1`, `A2`, `A3`, `Meta`*): [EventCreator3](#eventcreator3)`A1`, `A2`, `A3`, `Payload`, `Meta`



*Defined in core/createEvent/createEvent.ts:41*



Creates an event creator that accepts no arguments ([EmptyEventCreator](#emptyeventcreator)). Description is optional.

### Example

    import { createEvent } from 'stapp'
    const submit = createEvent('Form submit')
    
    store.dispatch(submit())


**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| description | `undefined`⎮`string`   |  Event description |





**Returns:** [EmptyEventCreator](#emptyeventcreator)



*Defined in core/createEvent/createEvent.ts:57*



Creates an event creator, that accepts single argument and passes it to an event as a payload ([EventCreator1](#eventcreator1)). Description is optional.

### Example

    import { createEvent } from 'stapp'
    const add = createEvent<number>('Increase')
    
    store.dispatch(add(1))
*__typeparam__*: Type of event meta



**Type parameters:**

#### Payload 

Type of event creator argument (and payload)

**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| description | `undefined`⎮`string`   |  Event description |





**Returns:** [EventCreator1](#eventcreator1)`Payload`, `Payload`



*Defined in core/createEvent/createEvent.ts:75*



Creates an event creator that accepts no arguments ([EventCreator0](#eventcreator0)).

### Example

    import { createEvent } from 'stapp'
    const addOne = createEvent('Add one', () => 1)
    
    store.dispatch(addOne())


**Type parameters:**

#### Payload 

Type of event payload

#### Meta 

Type of event meta

**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| description | `string`   |  Event description |
| payloadCreator | [PayloadTransformer0](#payloadtransformer0)`Payload`   |  Optional payload creator |
| metaCreator | [PayloadTransformer0](#payloadtransformer0)`Meta`   |  Optional meta creator |





**Returns:** [EventCreator0](#eventcreator0)`Payload`, `Meta`



*Defined in core/createEvent/createEvent.ts:98*



Creates an event creator that accepts and transforms single argument ([EventCreator1](#eventcreator1)).

### Example

    import { createEvent } from 'stapp'
    const addDouble = createEvent('Add doubled value', (x: number) => x * 2)
    
    addDouble(10).payload === 10 // true


**Type parameters:**

#### A1 

Type of event creator argument

#### Payload 

Type of event payload

#### Meta 

Type of event meta

**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| description | `string`   |  Event description |
| payloadCreator | [PayloadTransformer1](#payloadtransformer1)`A1`, `Payload`   |  Payload creator |
| metaCreator | [PayloadTransformer1](#payloadtransformer1)`A1`, `Meta`   |  Optional meta creator |





**Returns:** [EventCreator1](#eventcreator1)`A1`, `Payload`, `Meta`



*Defined in core/createEvent/createEvent.ts:122*



Creates an event creator that accepts and transforms two arguments ([EventCreator2](#eventcreator2)).

### Example

    import { createEvent } from 'stapp'
    const sum = createEvent('Combine values', (x: number, y: string) => x + Number(y))
    
    sum(10, '20').payload === 30 // true


**Type parameters:**

#### A1 

Type of event creator first argument

#### A2 

Type of event creator second argument

#### Payload 

Type of event payload

#### Meta 

Type of event meta

**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| description | `string`   |  Event description |
| payloadCreator | [PayloadTransformer2](#payloadtransformer2)`A1`, `A2`, `Payload`   |  Payload creator |
| metaCreator | [PayloadTransformer2](#payloadtransformer2)`A1`, `A2`, `Meta`   |  Optional meta creator |





**Returns:** [EventCreator2](#eventcreator2)`A1`, `A2`, `Payload`, `Meta`



*Defined in core/createEvent/createEvent.ts:147*



Creates an event creator that accepts and transforms three arguments ([EventCreator2](#eventcreator2)).

### Example

    import { createEvent } from 'stapp'
    const waaaat = createEvent('Combine values', (x: number, y: string, z: { n: number }) => (x + Number(y) - z.n).toString())
    
    waaaat(10, '20', { n: 100 }).payload === '-70' // true


**Type parameters:**

#### A1 

Type of event creator first argument

#### A2 

Type of event creator second argument

#### A3 

Type of event creator third argument

#### Payload 

Type of event payload

#### Meta 

Type of event meta

**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| description | `string`   |  Event description |
| payloadCreator | [PayloadTransformer3](#payloadtransformer3)`A1`, `A2`, `A3`, `Payload`   |  Payload creator |
| metaCreator | [PayloadTransformer3](#payloadtransformer3)`A1`, `A2`, `A3`, `Meta`   |  Optional meta creator |





**Returns:** [EventCreator3](#eventcreator3)`A1`, `A2`, `A3`, `Payload`, `Meta`





___

<a id="createfield"></a>

### «Const» createField

► **createField**State,Api(app: *[Stapp](#stapp)`State`, `Api`*): `StatelessComponent`.<[FieldProps](#fieldprops)>



*Defined in react/createField/createField.tsx:51*



Creates react form helpers

Form example
------------

     import { createForm } from 'stapp/lib/react'
     import someApp from '../myApps/app.js'
    
     const { Form, Field } = createForm(someApp)
    
     <Form>
       {
         ({
           handleSubmit,
           isReady,
           isValid
         }) => <form onSubmit={ handleSubmit }>
           <Field name='name'>
             {
               ({ input, meta }) => <React.Fragment>
                 <input { ...input} />
                 { meta.touched && meta.error && <span>{ meta.error }</span> }
               </React.Fragment>
             }
           </Field>
           <button
             type='submit'
             disabled={!isReady || !isValid}
            >
             Submit
            </button>
         </form>
       }
     </Form>
    

See more examples in the examples folder.


**Type parameters:**

#### State 
#### Api 
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| app | [Stapp](#stapp)`State`, `Api`   |  Stapp application |





**Returns:** `StatelessComponent`.<[FieldProps](#fieldprops)>





___

<a id="createform"></a>

### «Const» createForm

► **createForm**State,Api(app: *[Stapp](#stapp)`State`, `Api`*): `StatelessComponent`.<[FormProps](#formprops)>



*Defined in react/createForm/createForm.tsx:57*



Creates react form helpers

Form example
------------

     import { createForm } from 'stapp/lib/react'
     import someApp from '../myApps/app.js'
    
     const { Form, Field } = createForm(someApp)
    
     <Form>
       {
         ({
           handleSubmit,
           isReady,
           isValid
         }) => <form onSubmit={ handleSubmit }>
           <Field name='name'>
             {
               ({ input, meta }) => <React.Fragment>
                 <input { ...input} />
                 { meta.touched && meta.error && <span>{ meta.error }</span> }
               </React.Fragment>
             }
           </Field>
           <button
             type='submit'
             disabled={!isReady || !isValid}
            >
             Submit
            </button>
         </form>
       }
     </Form>
    

See more examples in the examples folder.


**Type parameters:**

#### State 
#### Api 
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| app | [Stapp](#stapp)`State`, `Api`   |  Stapp application |





**Returns:** `StatelessComponent`.<[FormProps](#formprops)>





___

<a id="createformbasereducers"></a>

### «Private»«Const» createFormBaseReducers

► **createFormBaseReducers**(initialState: *`any`*): `object`



*Defined in modules/formBase/reducers.ts:38*



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| initialState | `any`   |  - |





**Returns:** `object`





___

<a id="createreducer"></a>

### «Const» createReducer

► **createReducer**S(initialState: *`S`*, handlers?: *[EventHandlers](#eventhandlers)`S`, `any`*): [Reducer](#reducer)`S`



*Defined in core/createReducer/createReducer.ts:17*



Creates reducer with some additional methods


**Type parameters:**

#### S 

Reducer's state interface

**Parameters:**

| Param | Type | Default value | Description |
| ------ | ------ | ------ | ------ |
| initialState | `S`  | - |   Initial state for the reducer |
| handlers | [EventHandlers](#eventhandlers)`S`, `any`  |  {} |   Object with event types as keys and EventHandlers as values |





**Returns:** [Reducer](#reducer)`S`







___

<a id="createstatestreamenhancer"></a>

### «Private»«Const» createStateStreamEnhancer

► **createStateStreamEnhancer**State(rootEpic: *[Epic](#epic)`State`*): `object`



*Defined in epics/createStateStreamEnhancer/createStateStreamEnhancer.ts:23*



Used to pass a stream of state to the middleware. Also, allows to dispatch observables instead of events.


**Type parameters:**

#### State 

Application state shape

**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| rootEpic | [Epic](#epic)`State`   |  - |





**Returns:** `object`





___

<a id="defaultmergeprops"></a>

### «Private»«Const» defaultMergeProps

► **defaultMergeProps**(...args: *`any`[]*): `any`



*Defined in react/createConsume/createConsume.tsx:15*



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| args | `any`[]   |  - |





**Returns:** `any`





___

<a id="diffset"></a>

### «Private»«Const» diffSet

► **diffSet**T(setA: *`Set`.<`T`>*, setB: *`Set`.<`T`>*): `Set`.<`any`>



*Defined in helpers/diffSet/diffSet.ts:8*



Collects entries from a second set, that do not present in a first set


**Type parameters:**

#### T 
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| setA | `Set`.<`T`>   |  - |
| setB | `Set`.<`T`>   |  - |





**Returns:** `Set`.<`any`>
New set comprising found entries





___

<a id="falsify"></a>

### «Private»«Const» falsify

► **falsify**T(obj: *`T`*): `object`



*Defined in helpers/falsify/falsify.ts:5*



Changes all own values of a provided object to false


**Type parameters:**

#### T 
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| obj | `T`   |  - |





**Returns:** `object`





___

<a id="fieldselector"></a>

### «Const» fieldSelector

► **fieldSelector**(name: *`string`*): `function`



*Defined in modules/formBase/selectors.ts:24*



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| name | `string`   |  - |





**Returns:** `function`





___

<a id="formbase"></a>

### «Const» formBase

► **formBase**FormValues,ReadyKeys(config?: *`object`*): [Module](#module)`__type`, [FormBaseState](#formbasestate)`FormValues`, `ReadyKeys`



*Defined in modules/formBase/formBase.ts:13*



Base form module


**Type parameters:**

#### FormValues 

Application state shape

#### ReadyKeys :  `string`

List of fields used for readiness reducer

**Parameters:**

| Param | Type | Default value | Description |
| ------ | ------ | ------ | ------ |
| config | `object`  |  {} |   - |





**Returns:** [Module](#module)`__type`, [FormBaseState](#formbasestate)`FormValues`, `ReadyKeys`





___

<a id="getdisplayname"></a>

### «Private»«Const» getDisplayName

► **getDisplayName**(Component: *`ComponentType`.<`any`>*): `string`



*Defined in react/helpers/getDisplayName.ts:7*



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| Component | `ComponentType`.<`any`>   |  - |





**Returns:** `string`





___

<a id="getepics"></a>

### «Private»«Const» getEpics

► **getEpics**(modules: *`Array`.<[Module](#module)`any`, `any`>*): `function`[]



*Defined in helpers/getEpics/getEpics.ts:52*



Gets epics from an array of passed modules


**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| modules | `Array`.<[Module](#module)`any`, `any`>   |  - |





**Returns:** `function`[]





___

<a id="geteventtype"></a>

### «Private» getEventType

► **getEventType**(eventCreator: *[AnyEventCreator](#anyeventcreator)⎮`string`*): `string`



*Defined in helpers/getEventType/getEventType.ts:10*



Gets type from an eventCreator


**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| eventCreator | [AnyEventCreator](#anyeventcreator)⎮`string`   |  - |





**Returns:** `string`





___

<a id="getevents"></a>

### «Private»«Const» getEvents

► **getEvents**(modules: *`Array`.<[Module](#module)`any`, `any`, `any`>*): `Object`



*Defined in helpers/getEvents/getEvents.ts:11*



Gets event creators from modules


**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| modules | `Array`.<[Module](#module)`any`, `any`, `any`>   |  - |





**Returns:** `Object`





___

<a id="getinitialstate"></a>

### «Const» getInitialState

► **getInitialState**(reducer: *[Reducer](#reducer)`any`*): `any`



*Defined in helpers/testHelpers/getInitialState/getInitialState.ts:3*



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| reducer | [Reducer](#reducer)`any`   |  - |





**Returns:** `any`





___

<a id="getmodules"></a>

### «Private»«Const» getModules

► **getModules**Extra(maybeModules?: *`Array`.<[AnyModule](#anymodule)`Extra`, `any`, `any`>*, extraArgument: *`Extra`*): `object`[]



*Defined in helpers/getModules/getModules.ts:16*



Collects modules and checks dependencies


**Type parameters:**

#### Extra 

Module dependency

**Parameters:**

| Param | Type | Default value | Description |
| ------ | ------ | ------ | ------ |
| maybeModules | `Array`.<[AnyModule](#anymodule)`Extra`, `any`, `any`>  |  [] |   - |
| extraArgument | `Extra`  | - |   Dependency injection |





**Returns:** `object`[]





___

<a id="getpersistepics"></a>

### «Private»«Const» getPersistEpics

► **getPersistEpics**State(__namedParameters: *`object`*): `object`



*Defined in modules/persist/persist.ts:33*



**Type parameters:**

#### State 
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| __namedParameters | `object`   |  - |





**Returns:** `object`





___

<a id="getreducer"></a>

### «Private»«Const» getReducer

► **getReducer**(modules: *`Array`.<[Module](#module)`any`, `any`, `any`>*): [Reducer](#reducer)`any`



*Defined in helpers/getReducer/getReducer.ts:25*



Creates root reducer with some superpowers. And remember, Pete, great power comes with great responsibility.


**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| modules | `Array`.<[Module](#module)`any`, `any`, `any`>   |  - |





**Returns:** [Reducer](#reducer)`any`





___

<a id="has"></a>

### «Private»«Const» has

► **has**P,O(prop: *`P`*, obj: *`any`*): `boolean`



*Defined in helpers/has/has.ts:9*



Returns whether or not an object has an own property with the specified name


**Type parameters:**

#### P :  `string`
#### O :  `object`
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| prop | `P`   |  The name of the property to check for. |
| obj | `any`   |  The object to query. |





**Returns:** `boolean`
Whether the property exists.





___

<a id="identity"></a>

### «Private»«Const» identity

► **identity**T(x: *`T`*, ...other: *`any`[]*): `T`



*Defined in helpers/identity/identity.ts:9*



Returns passed argument


**Type parameters:**

#### T 
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| x | `T`   |  The value to return. |
| other | `any`[]   |  Ignored params |





**Returns:** `T`
The input value





___

<a id="isarray"></a>

### «Private»«Const» isArray

► **isArray**T(a: *`any`*): `boolean`



*Defined in helpers/isArray/isArray.ts:7*



Checks if provided argument is an array


**Type parameters:**

#### T 
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| a | `any`   |  - |





**Returns:** `boolean`





___

<a id="isdirtyselector"></a>

### «Const» isDirtySelector

► **isDirtySelector**(): `function``object`



*Defined in modules/formBase/selectors.ts:16*





**Returns:** `function``object`





___

<a id="iserror"></a>

### «Private»«Const» isError

► **isError**(a: *`any`*): `boolean`



*Defined in core/createEvent/createEvent.ts:27*



Checks if provided value is an instance of Error


**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| a | `any`   |  Любые данные |





**Returns:** `boolean`





___

<a id="isevent"></a>

### «Private»«Const» isEvent

► **isEvent**P,M(arg?: *`any`*): `boolean`



*Defined in helpers/isEvent/isEvent.ts:11*



Checks if provided argument is a valid Redux action / event


**Type parameters:**

#### P 
#### M 
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| arg | `any`   |  any value |





**Returns:** `boolean`





___

<a id="ispristineselector"></a>

### «Const» isPristineSelector

► **isPristineSelector**State(state: *`State`*): `boolean`



*Defined in modules/formBase/selectors.ts:22*



**Type parameters:**

#### State :  [FormBaseState](#formbasestate)
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| state | `State`   |  - |





**Returns:** `boolean`





___

<a id="isreadyselector"></a>

### «Const» isReadySelector

► **isReadySelector**(): `function``object`



*Defined in modules/formBase/selectors.ts:10*





**Returns:** `function``object`





___

<a id="isvalidselector"></a>

### «Const» isValidSelector

► **isValidSelector**(): `function``object`



*Defined in modules/formBase/selectors.ts:4*





**Returns:** `function``object`





___

<a id="mapmodule"></a>

### «Private»«Const» mapModule

► **mapModule**Api,State,Full(module: *[Module](#module)`Api`, `State`, `Full`*): [Epic](#epic)`State`



*Defined in helpers/getEpics/getEpics.ts:21*



Gets epics from modules Incoming module should be guaranteed to have an epic In development mode extends event meta with info about module name to keep track on it in devtools


**Type parameters:**

#### Api :  [EventCreators](#eventcreators)

Event creators

#### State 

Module state

#### Full :  `Partial`.<`State`>

Expected full state of application

**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| module | [Module](#module)`Api`, `State`, `Full`   |  - |





**Returns:** [Epic](#epic)`State`
epic





___

<a id="mapobject"></a>

### «Private»«Const» mapObject

► **mapObject**El,Result(fn: *`function`*, o: *`object`*): `object`



*Defined in modules/formBase/reducers.ts:22*



**Type parameters:**

#### El 
#### Result 
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| fn | `function`   |  - |
| o | `object`   |  - |





**Returns:** `object`





___

<a id="merge"></a>

### «Private»«Const» merge

► **merge**F,T,O(fields: *`F`[]*, objects: *`O`[]*): `T`



*Defined in helpers/merge/merge.ts:23*



Merges values of provided objects into one


**Type parameters:**

#### F :  `string`
#### T 
#### O :  `object`
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| fields | `F`[]   |  array of fields names |
| objects | `O`[]   |  - |





**Returns:** `T`





___

<a id="persist"></a>

### «Const» persist

► **persist**State(config: *[PersistConfig](#persistconfig)`State`*): [Module](#module)`object`, `__type`



*Defined in modules/persist/persist.ts:82*



Persistence module.


**Type parameters:**

#### State 

Application state shape

**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| config | [PersistConfig](#persistconfig)`State`   |  Configuration object |





**Returns:** [Module](#module)`object`, `__type`
Module






___

<a id="rendercomponent"></a>

### «Const» renderComponent

► **renderComponent**(props: *[RenderProps](#renderprops)`any`*, passedProps: *`any`*, name: *`string`*): `ReactElement`.<`any`>⎮`null`



*Defined in react/helpers/renderComponent.ts:5*



**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| props | [RenderProps](#renderprops)`any`   |  - |
| passedProps | `any`   |  - |
| name | `string`   |  - |





**Returns:** `ReactElement`.<`any`>⎮`null`





___

<a id="run"></a>

### «Private»«Const» run

► **run**Payload,Result(params: *`Payload`*, effect: *[EffectFn](#effectfn)`Payload`, `Result`*): `Promise`.<`Result`>



*Defined in core/createEffect/createEffect.ts:18*



**Type parameters:**

#### Payload 
#### Result 
**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| params | `Payload`   |  - |
| effect | [EffectFn](#effectfn)`Payload`, `Result`   |  - |





**Returns:** `Promise`.<`Result`>





___

<a id="select"></a>

###  select

► **select**Payload,Meta(eventCreator: *`string`⎮[AnyEventCreator](#anyeventcreator)`Payload`, `Meta`*, source$: *`Observable`.<[Event](#event)`any`, `any`>*): `Observable`.<[Event](#event)`Payload`, `Meta`>



*Defined in epics/select/select.ts:16*



Filters stream of events by type of provided event creator


**Type parameters:**

#### Payload 

Event payload

#### Meta 

Event meta data

**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| eventCreator | `string`⎮[AnyEventCreator](#anyeventcreator)`Payload`, `Meta`   |  Any event creator |
| source$ | `Observable`.<[Event](#event)`any`, `any`>   |  Stream of events |





**Returns:** `Observable`.<[Event](#event)`Payload`, `Meta`>
filtered stream of provided event type






___

<a id="selectarray"></a>

###  selectArray

► **selectArray**(eventCreators: *`Array`.<[AnyEventCreator](#anyeventcreator)⎮`string`>*, source$: *`Observable`.<[Event](#event)`any`, `any`>*): `Observable`.<[Event](#event)`any`, `any`>



*Defined in epics/select/select.ts:31*



Filters stream of events by provided event types


**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| eventCreators | `Array`.<[AnyEventCreator](#anyeventcreator)⎮`string`>   |  Array of any event creators and/or strings representing event types |
| source$ | `Observable`.<[Event](#event)`any`, `any`>   |  Stream of events |





**Returns:** `Observable`.<[Event](#event)`any`, `any`>
filtered stream of provided event types






___

<a id="uniqueid"></a>

### «Private»«Const» uniqueId

► **uniqueId**(): `number`



*Defined in helpers/uniqueId/uniqueId.ts:12*



Generates a unique ID.




**Returns:** `number`
Returns the unique ID.





___

<a id="usereduxenhancer"></a>

### «Private»«Const» useReduxEnhancer

► **useReduxEnhancer**(): `boolean`



*Defined in core/createApp/createApp.ts:22*





**Returns:** `boolean`







___

<a id="whenready"></a>

### «Const» whenReady

► **whenReady**(): `Promise`.<`object`>



*Defined in helpers/awaitStore/awaitStore.ts:24*



Returns promise, that resolves when all stores are ready. Primarily used for SSR.




**Returns:** `Promise`.<`object`>





___


<a id="consumerproptypes"></a>

## Object literal: consumerPropTypes



<a id="consumerproptypes.children"></a>

###  children

**●  children**:  *`Requireable`.<`any`>*  =  renderPropType

*Defined in react/createConsumer/createConsumer.ts:31*





___
<a id="consumerproptypes.component"></a>

###  component

**●  component**:  *`Requireable`.<`any`>*  =  renderPropType

*Defined in react/createConsumer/createConsumer.ts:32*





___
<a id="consumerproptypes.mapapi"></a>

###  mapApi

**●  mapApi**:  *`Requireable`.<`any`>*  =  selectorType

*Defined in react/createConsumer/createConsumer.ts:34*





___
<a id="consumerproptypes.mapstate"></a>

###  mapState

**●  mapState**:  *`Requireable`.<`any`>*  =  selectorType

*Defined in react/createConsumer/createConsumer.ts:33*





___
<a id="consumerproptypes.render"></a>

###  render

**●  render**:  *`Requireable`.<`any`>*  =  renderPropType

*Defined in react/createConsumer/createConsumer.ts:30*





___

<a id="loggermodule"></a>

## Object literal: loggerModule



<a id="loggermodule.name"></a>

###  name

**●  name**:  *`string`*  = "logger"

*Defined in helpers/testHelpers/loggerModule/loggerModule.ts:8*





___
<a id="loggermodule.reducers"></a>

###  reducers

** reducers**:  *`object`* 

*Defined in helpers/testHelpers/loggerModule/loggerModule.ts:9*




<a id="loggermodule.reducers.eventlog"></a>

###  eventLog

► **eventLog**(state?: *`Array`.<[Event](#event)`any`, `any`>*, event: *[Event](#event)`any`, `any`*): `object`[]



*Defined in helpers/testHelpers/loggerModule/loggerModule.ts:10*



**Parameters:**

| Param | Type | Default value | Description |
| ------ | ------ | ------ | ------ |
| state | `Array`.<[Event](#event)`any`, `any`>  |  [] |   - |
| event | [Event](#event)`any`, `any`  | - |   - |





**Returns:** `object`[]





___

___


