


#  stapp

## Index

### Type aliases

* [Effect](#effect)
* [EffectFn](#effectfn)
* [Epic](#epic)
* [Event](#event)
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
* [PayloadTransformer0](#payloadtransformer0)
* [PayloadTransformer1](#payloadtransformer1)
* [PayloadTransformer2](#payloadtransformer2)
* [PayloadTransformer3](#payloadtransformer3)
* [PersistConfig](#persistconfig)
* [PersistStorage](#persiststorage)
* [Reducer](#reducer)
* [Stapp](#stapp)


### Variables

* [FORM_BASE](#form_base)
* [dangerouslyReplaceState](#dangerouslyreplacestate)
* [dangerouslyResetState](#dangerouslyresetstate)
* [epicEnd](#epicend)
* [resetForm](#resetform)
* [setActive](#setactive)
* [setError](#seterror)
* [setReady](#setready)
* [setTouched](#settouched)
* [setValue](#setvalue)
* [submit](#submit)


### Functions

* [combineEpics](#combineepics)
* [createApp](#createapp)
* [createConsume](#createconsume)
* [createConsumer](#createconsumer)
* [createEffect](#createeffect)
* [createEvent](#createevent)
* [createField](#createfield)
* [createForm](#createform)
* [createReducer](#createreducer)
* [fieldSelector](#fieldselector)
* [formBase](#formbase)
* [getInitialState](#getinitialstate)
* [isDirtySelector](#isdirtyselector)
* [isPristineSelector](#ispristineselector)
* [isReadySelector](#isreadyselector)
* [isValidSelector](#isvalidselector)
* [persist](#persist)
* [renderComponent](#rendercomponent)
* [select](#select)
* [selectArray](#selectarray)
* [whenReady](#whenready)



---
# Type aliases








<a id="effect"></a>

###  Effect

**Τ Effect**:  *`object`* 



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








**Returns:** `string`





 use : function
► **use**(effectFn: *[EffectFn](#effectfn)`Payload`, `Result`*): [Effect](#effect)`Payload`, `Result`






**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| effectFn | [EffectFn](#effectfn)`Payload`, `Result`   |  - |





**Returns:** [Effect](#effect)`Payload`, `Result`







___

<a id="effectfn"></a>

###  EffectFn

**Τ EffectFn**:  *`function`* 




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


<a id="epic"></a>

###  Epic

**Τ Epic**:  *[EventEpic](#eventepic)`any`, `any`, `State`* 




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



#### Type declaration




 error: `boolean`






 meta: `Meta`






 payload: `Payload`






 type: `string`







___





<a id="eventcreators"></a>

###  EventCreators

**Τ EventCreators**:  *`object`* 




An object with various event creators as values

#### Type declaration


[K: `string`]: [AnyEventCreator](#anyeventcreator)






___

<a id="eventepic"></a>

###  EventEpic

**Τ EventEpic**:  *`function`* 




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




An object with various event handlers as values

#### Type declaration





___

<a id="fieldapi"></a>

###  FieldApi

**Τ FieldApi**:  *`object`* 



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






___

<a id="formapi"></a>

###  FormApi

**Τ FormApi**:  *`object`* 



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






___

<a id="module"></a>

###  Module

**Τ Module**:  *`object`* 




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


<a id="payloadtransformer0"></a>

###  PayloadTransformer0

**Τ PayloadTransformer0**:  *`function`* 




#### Type declaration
►(): `Payload`





**Returns:** `Payload`






___

<a id="payloadtransformer1"></a>

###  PayloadTransformer1

**Τ PayloadTransformer1**:  *`function`* 




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



#### Type declaration



 getItem : function
► **getItem**(key: *`string`*): `string`⎮`null`⎮`undefined`






**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| key | `string`   |  - |





**Returns:** `string`⎮`null`⎮`undefined`





 removeItem : function
► **removeItem**(key: *`string`*): `void`






**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| key | `string`   |  - |





**Returns:** `void`





 setItem : function
► **setItem**(key: *`string`*, data: *`string`*): `void`






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


<a id="stapp"></a>

###  Stapp

**Τ Stapp**:  *`object`* 




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



<a id="form_base"></a>

### «Const» FORM_BASE

**●  FORM_BASE**:  *`string`*  =  `${APP_KEY}/formBase`






___




<a id="dangerouslyreplacestate"></a>

### «Const» dangerouslyReplaceState

**●  dangerouslyReplaceState**:  *[EventCreator1](#eventcreator1)`any`*  =  createEvent<any>(
  `${APP_KEY}: Replace state`
)




Can be used to replace state completely




___


<a id="dangerouslyresetstate"></a>

### «Const» dangerouslyResetState

**●  dangerouslyResetState**:  *[EmptyEventCreator](#emptyeventcreator)*  =  createEvent(`${APP_KEY}: Reset state`)




Can be used to reinitialize state




___


<a id="epicend"></a>

### «Const» epicEnd

**●  epicEnd**:  *[EmptyEventCreator](#emptyeventcreator)*  =  createEvent(`${APP_KEY}: Epic enc`)




Event signaling about epic completing




___






<a id="resetform"></a>

### «Const» resetForm

**●  resetForm**:  *`object``function`*  =  createEvent(`${FORM_BASE}: Reset form state`)




Used to reset form state




___


<a id="setactive"></a>

### «Const» setActive

**●  setActive**:  *`object``function`*  =  createEvent<string | null>(`${FORM_BASE}: Set field as active`)




Set field as active




___

<a id="seterror"></a>

### «Const» setError

**●  setError**:  *`object``function`*  =  createEvent<{ [K: string]: string }>(`${FORM_BASE}: Set field error`)




Used to set errors for fields




___

<a id="setready"></a>

### «Const» setReady

**●  setReady**:  *`object``function`*  =  createEvent<{ [K: string]: boolean }>(`${FORM_BASE}: Set readiness`)




Used to set readiness state




___

<a id="settouched"></a>

### «Const» setTouched

**●  setTouched**:  *`object``function`*  =  createEvent<{ [K: string]: boolean }>(
  `${FORM_BASE}: Set field as touched`
)




Used to set field as touched




___

<a id="setvalue"></a>

### «Const» setValue

**●  setValue**:  *`object``function`*  =  createEvent<{ [K: string]: any }>(`${FORM_BASE}: Set field value`)




Used to set values for fields




___

<a id="submit"></a>

### «Const» submit

**●  submit**:  *`object``function`*  =  createEvent(`${FORM_BASE}: Submit`, () => undefined)




Used to indicate form submission




___


# Functions




<a id="combineepics"></a>

### «Const» combineEpics

► **combineEpics**S1,S2,S3,S4,S5,S6,S7,S8,S9,S10,State(epics: *[[Epic](#epic)`S1`,[Epic](#epic)`S2`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`,[Epic](#epic)`S5`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`,[Epic](#epic)`S5`,[Epic](#epic)`S6`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`,[Epic](#epic)`S5`,[Epic](#epic)`S6`,[Epic](#epic)`S7`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`,[Epic](#epic)`S5`,[Epic](#epic)`S6`,[Epic](#epic)`S7`,[Epic](#epic)`S8`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`,[Epic](#epic)`S5`,[Epic](#epic)`S6`,[Epic](#epic)`S7`,[Epic](#epic)`S8`,[Epic](#epic)`S9`]⎮[[Epic](#epic)`S1`,[Epic](#epic)`S2`,[Epic](#epic)`S3`,[Epic](#epic)`S4`,[Epic](#epic)`S5`,[Epic](#epic)`S6`,[Epic](#epic)`S7`,[Epic](#epic)`S8`,[Epic](#epic)`S9`,[Epic](#epic)`S10`]*): [Epic](#epic)`State`






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




<a id="createapp"></a>

###  createApp

► **createApp**A1,S1,E1,A2,S2,E2,A3,S3,E3,A4,S4,E4,A5,S5,E5,A6,S6,E6,A7,S7,E7,A8,S8,E8,A9,S9,E9,A10,S10,E10,Extra,State,Api(config: *`object`*): [Stapp](#stapp)`State`, `Api`






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


<a id="createconsume"></a>

### «Const» createConsume

► **createConsume**State,Api(app: *[Stapp](#stapp)`State`, `Api`*): [ConsumerHoc](interfaces/consumerhoc.md)`State`, `Api`






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


<a id="createreducer"></a>

### «Const» createReducer

► **createReducer**S(initialState: *`S`*, handlers?: *[EventHandlers](#eventhandlers)`S`, `any`*): [Reducer](#reducer)`S`






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





<a id="fieldselector"></a>

### «Const» fieldSelector

► **fieldSelector**(name: *`string`*): `function`






**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| name | `string`   |  - |





**Returns:** `function`





___

<a id="formbase"></a>

### «Const» formBase

► **formBase**FormValues,ReadyKeys(config?: *`object`*): [Module](#module)`__type`, [FormBaseState](#formbasestate)`FormValues`, `ReadyKeys`






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





<a id="getinitialstate"></a>

### «Const» getInitialState

► **getInitialState**(reducer: *[Reducer](#reducer)`any`*): `any`






**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| reducer | [Reducer](#reducer)`any`   |  - |





**Returns:** `any`





___







<a id="isdirtyselector"></a>

### «Const» isDirtySelector

► **isDirtySelector**(): `function``object`








**Returns:** `function``object`





___



<a id="ispristineselector"></a>

### «Const» isPristineSelector

► **isPristineSelector**State(state: *`State`*): `boolean`






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








**Returns:** `function``object`





___

<a id="isvalidselector"></a>

### «Const» isValidSelector

► **isValidSelector**(): `function``object`








**Returns:** `function``object`





___




<a id="persist"></a>

### «Const» persist

► **persist**State(config: *[PersistConfig](#persistconfig)`State`*): [Module](#module)`object`, `__type`






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






**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| props | [RenderProps](#renderprops)`any`   |  - |
| passedProps | `any`   |  - |
| name | `string`   |  - |





**Returns:** `ReactElement`.<`any`>⎮`null`





___


<a id="select"></a>

###  select

► **select**Payload,Meta(eventCreator: *`string`⎮[AnyEventCreator](#anyeventcreator)`Payload`, `Meta`*, source$: *`Observable`.<[Event](#event)`any`, `any`>*): `Observable`.<[Event](#event)`Payload`, `Meta`>






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






Filters stream of events by provided event types


**Parameters:**

| Param | Type | Description |
| ------ | ------ | ------ |
| eventCreators | `Array`.<[AnyEventCreator](#anyeventcreator)⎮`string`>   |  Array of any event creators and/or strings representing event types |
| source$ | `Observable`.<[Event](#event)`any`, `any`>   |  Stream of events |





**Returns:** `Observable`.<[Event](#event)`any`, `any`>
filtered stream of provided event types






___



<a id="whenready"></a>

### «Const» whenReady

► **whenReady**(): `Promise`.<`object`>






Returns promise, that resolves when all stores are ready. Primarily used for SSR.




**Returns:** `Promise`.<`object`>





___



