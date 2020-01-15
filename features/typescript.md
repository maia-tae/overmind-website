# Typescript

Overmind is written in Typescript and it is written with a focus on you dedicating as little time as possible to help Typescript understand what your app is all about. Typescript will spend a lot more time helping you. If you are not a Typescript developer Overmind is a really great project to start learning it as you will get the most out of the little typing you have to do.

## Two typing approaches

### 1. Declare module

The most straightforward way to type your application is to use the **declare module** approach. This will work for most applications, but might make you feel uncomfortable as a hardcore Typescripter. The reason is that we are overriding an internal type, meaning that you can only have one instance of Overmind running inside your application.

{% tabs %}
{% tab title="overmind/index.ts" %}
```typescript
import { IConfig } from 'overmind'

const config = {}

declare module 'overmind' {
  // tslint:disable:interface-name
  interface Config extends IConfig<typeof config> {}
}
```
{% endtab %}
{% endtabs %}

Now you can import any type directly from Overmind and it will understand the configuration of your application. Even the operators are typed.

```typescript
import {
  Action,
  Operator,
  Derive,
  pipe,
  map,
  filter,
  ...
} from 'overmind'
```

### 2. Explicit typing

You can also explicitly type your application. This gives more flexibility.

{% tabs %}
{% tab title="overmind/index.ts" %}
```typescript
import {
  IConfig,
  IOnInitialize,
  IAction,
  IOperator,
  IDerive,
  IState
} from 'overmind'

export const config = {}

export interface Config extends IConfig<typeof config> {}

export interface OnInitialize extends IOnInitialize<Config> {}

export interface Action<Input = void, Output = void> extends IAction<Config, Input, Output> {}

export interface AsyncAction<Input = void, Output = void> extends IAction<Config, Input, Promise<Output>> {}

export interface Operator<Input = void, Output = Input> extends IOperator<Config, Input, Output> {}

export interface Derive<Parent extends IState, Output> extends IDerive<Config, Parent, Output> {}
```
{% endtab %}
{% endtabs %}

You only have to set up these types once, where you bring your configuration together. That means if you use multiple namespaced configuration you still only create one set of types, as shown above.

Now you only have to make sure that you import your types from this file, instead of directly from the Overmind package.

{% tabs %}
{% tab title="overmind/actions.ts" %}
```typescript
import { Action } from './'

const doThis: Action = () => {}
```
{% endtab %}

{% tab title="overmind/someNamespace/actions.ts" %}
```typescript
import { Action } from '../'

const doThis: Action = () => {}
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
The Overmind documentation is written for implicit typing. That means whenever you see a type import directly from the Overmind package, you should rather import from your own defined types.
{% endhint %}

## Linting

When you are using TSLint it is important that you use the official [MICROSOFT EXTENSION](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-typescript-tslint-plugin) for VS Code.

## Actions

The action type takes either an input type, an output type, or both.

```typescript
import { Action } from 'overmind'

export const noArgAction: Action = (context, value) => {
  value // this becomes "void"
}

export const argAction: Action<string> = (context, value) => {
  value // this becomes "string"
}

export const noArgWithReturnTypeAction: Action<void, string> = (context, value) => {
  value // this becomes "void"

  return 'foo'
}

export const argWithReturnTypeAction: Action<string, string> = (context, value) => {
  value // this becomes "string"

  return value + '!!!'
}
```

You also have an **async** version of this type. You use this when you want to define an **async** function, which implicitly returns a promise, or a function that explicitly returns a promise.

```typescript
import { AsyncAction } from 'overmind'

export const noArgAction: AsyncAction = async (context, value) => {
  value // this becomes "void"
}

export const argAction: AsyncAction<string> = async (context, value) => {
  value // this becomes "string"
}

export const noArgWithReturnTypeAction: AsyncAction<void, string> = async (context, value) => {
  value // this becomes "void"

  return 'foo'
} // returns Promise<string>

export const argWithReturnTypeAction: AsyncAction<string, string> = (context, value) => {
  value // this becomes "string"

  return Promise.resolve(value + '!!!')
} // returns Promise<string>
```

## Operators

Operators is like the **Action** type: it can take an optional input, but it always produces an output. By default the output of an operator is the same as the input.

{% tabs %}
{% tab title="overmind/operators.ts" %}
```typescript
import { Operator, mutate, filter, map } from 'overmind'

// You do not need to define any types, which means it defaults
// its input and output to "void"
export const changeSomeState: () => Operator = () =>
  mutate(function changeSomeState({ state }) {
    state.foo = 'bar'
  })

// The second type argument is not set, but will default to "User"
// The output is the same as the input
export const filterAwesomeUser: () => Operator<User> = () =>
  filter(function filterAwesomeUser(_, user) {
    return user.isAwesome
  })

// "map" produces a new output so we define that as the second
// type argument
export const toNumber: () => Operator<string, number> = () =>
  map(function toNumber(_, value) { 
    return Number(value)
  })
```
{% endtab %}
{% endtabs %}

The **Operator** type is used to type all operators. The type arguments you give to **Operator** have to match the specific operator you use though. So for example if you type a **mutate** operator with a different output than the input:

```typescript
import { Operator, mutate } from 'overmind'

export const doThis: () => Operator<string, number> = () => 
  mutate(function doThis() {

  })
```

Typescript yells at you, because this operator just passes the value straight through.

Typically you do not think about this and Typescript rather yells at you when the value you are passing through your operators is not matching up.

### Generic input

You might create an operator that does not care about its input. For example:

{% tabs %}
{% tab title="overmind/operators.ts" %}
```typescript
import { Operator, mutate } from 'overmind'

export const doSomething: () => Operator = () =>
  mutate(function doSomething({ state }) {
    state.foo = 'bar'
  })
```
{% endtab %}

{% tab title="overmind/actions.ts" %}
```typescript
import { Operator, pipe, action } from 'overmind'
import * as o from './operators'

export const setInput: Operator<string> = pipe(
  o.doSomething(),
  o.setValue()
)
```
{% endtab %}
{% endtabs %}

Composing **doSomething** into the **pipe** gives an error, cause the action is typed with a **string** input, but the **doSomething** operator is typed with **void**.

To fix this we just add a generic type to the definition of our operator:

{% tabs %}
{% tab title="overmind/operators.ts" %}
```typescript
import { Operator, mutate } from 'overmind'

export const doSomething: <T>() => Operator<T> = () =>
  mutate(function doSomething({ state }) {
    state.foo = 'bar'
  })
```
{% endtab %}
{% endtabs %}

Now Typescript infers the input type of the operator and passes it along.

### Partial input

For example:

{% tabs %}
{% tab title="overmind/operators.ts" %}
```typescript
import { Operator, filter } from 'overmind'

export const filterAwesome: () => Operator<{ isAwesome: boolean }> = () =>
  filter(function filterAwesome(_, somethingAwesome) {
    return somethingAwesome.isAwesome
  })
```
{% endtab %}

{% tab title="overmind/actions.ts" %}
```typescript
import { Operator, pipe, action } from 'overmind'
import * as o from './operators'
import { User } from './state'

export const clickedUser: Operator<User> = pipe(
  o.filterAwesome(),
  o.handleAwesomeUser()
)
```
{% endtab %}
{% endtabs %}

Now the _input_ is actually okay, because `{ isAwesome: boolean }` matches the **User** type, but we are also now saying that the type of _output_ will be `{ isAwesome: boolean }`, which does not match the **User** type required by **handleAwesomeUser**.

To fix this we again infer the type, but using **extends** to indicate that we do have a requirement to the type it should pass through:

{% tabs %}
{% tab title="overmind/operators.ts" %}
```typescript
import { Operator, filter } from 'overmind'

export const filterAwesome: <T extends { isAwesome: boolean }>() => Operator<T> =
  () => filter(function filterAwesome(_, somethingAwesome) {
    return somethingAwesome.isAwesome
  })
```
{% endtab %}
{% endtabs %}

That means this operator can handle any type that matches an **isAwesome** property, though will pass the original type through.
