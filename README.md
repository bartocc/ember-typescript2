# Ember CLI with TypeScript 2.3+ (EXPERIMENTAL)

This README outlines the details of collaborating on this Ember addon (_see: "Contributing"_)

For more information on using ember-cli, visit
[https://ember-cli.com/](https://ember-cli.com/).


## WARNING: EXPERIMENTAL ADDON

Please note that the type declarations are in an early stage and far from complete. The addon itself hasn't been fully tested and won't catch all errors in your code (although it will help a lot!). Also the way the Broccoli funnel is set up is still kind of hacky.

> If you want to help out with this project, pull requests are welcomed!


# What's included?

### Ember Interfaces and Types
A rewrite of Ember 2.x types and interfaces, based on Typescript 2.3+ features such as:

- indexed signatures
- _ThisType_
- _keyof_ and _lookup types_

### Out of the box .ts support in Ember CLI build

- This addon runs Typescript during the Ember CLI build process (using `broccoli-typescript-compiler`)
- All `.ts` and `.js` scripts in the *app/* folder are type checked and compiled via `ember serve`
- Any type errors in your app are displayed in the Ember CLI console

### Dependencies:

- typescript@2.3.4
- broccoli-typescript-compiler

---

# Installation

To install this addon and start using Typescript in your Ember project:

```
$ cd my-ember-app/
$ ember install ember-typescript2
```

This will install a *tsconfig.json*, and some extendable interfaces in *app/types/application.d.ts* (see section ModelTypeIndex below).

### Usage 

Run **ember serve** and you should see the result of Typescript type checking:

```
$ cd my-ember-app/
$ ember serve
```

### IDE integration

If you are using Visual Studio Code, you can enable its *Typescript Language mode* while editing .js files:

Menu: _View > Command Pallette .... > Language Mode > Typescript_

### Running TypeScripts' compiler (`tsc`) standalone

Note that for your convenience, a tsconfig.json is installed in
your projects root. This allows for better IDE integration, and you can also 
run Typescript in standalone mode on your project:

Install Typescript globally:

```
$ npm install -g typescript
```

Now run **tsc** in your project, and get pretty printed type errors:

```
$ cd my-ember-app/
$ tsc
```



### Configuration: your application.d.ts

To get Typechecking working for your models and services, have a look at
*app/types/application.d.ts* (automatically generated by ember-typescript2)

_For more information, see the section **Advanced configuration** below._

----

# TypeScript features bundled in this addon:

For some Ember application code, the bundled Typescript 2.3 definitions work out of the box. This means you will now get type checking for things like:


## Ease of use, no need to rewrite any code:

This addon was created in order to experiment with TypeScript inside of Ember applications.

The goal is to have a simple drop-in addon which enables typechecking across any Ember app, without a need to rewrite any code.

While some minor configuration of the **app/types/application.d.ts** file is recommended (see section Configuration), this means you can keep writing your app in the conventional Ember CLI style, and get TypeScript type checking out of the box:

````typescript
import Ember from 'ember';

export default Ember.Route.extend({
  setupController(controller, model) {
    var user = this.store.createRecord('user');
    user.set('nonExistent', true);
              ^^^^^^^^^^^
              type error because property does not exist on type User
  }
});
````

> Please note that TypeScript needs to know about your Models and Services. Please refer to the section ModelTypeIndex and ServiceTypeIndex


## Type of `this` in methods (via ThisType):

As of Typescript 2.3, it is now possible to properly set the `this` context within Ember methods. Where the following code would previously break down, Typescript can now understand the following Ember code:

```typescript
export default Ember.Component.extend({

  color: 'red',

  logColor() {
    let color = this.get('color');
    //    ^-- the type of `color` is "string"

    console.log(`Color: ${color}`);
  }
});
```

## Type checking Ember.Object (get and set):

[TypeScript 2.1](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-1.html) comes with `keyof` and `lookup types`, this enables TypeScript to statically check if an object has certain properties, and also retrieve the type of those properties:

```typescript
let MyObj = Ember.Object.create({
  color: "red"
})

var color = MyObj.get('color');
//    ^-- color is of type "string"

var hasShadow = MyObj.get('hasShadow');
//                        ^^^^^^^^^^
//                        Type error: `hasShadow` is not a property of `MyObj`
```


`object.get(key)`
  - existence of `key` on `object` is checked
  - the returned value has its type information preserved
`object.set(key, val)`
  - existence of `key` on `object` is checked
  - the type of `val` is checked against the type of `object.key`


## Ember Data: auto generate model interfaces:

TypeScript's type inference features makes it possible to generate types for your Ember models and Services:

`this.store.createRecord('user', /* ... */)`
  - the existence of a model with name `user` is checked
  - the created record will be of type User (NOTE: see section MoodelTypeIndex below to set this up)
  - model.get(key) works (as model extends from Ember.Object)
  
> WARNING: computed properties aren't TypeSafe, and you won't get a Type error if you try to call set() on a computed property. However if you define your computed properties as readOnly(), Ember will give you a runtime error, which is still preferable above a silent failure
  
## Other features:

- type errors when using deprecated Ember feature flags like MODEL_FACTORY_INJECTIONS (needs to be extended)
- Ember.String types (work in progress)
- Ember.Route hooks such as `model` and `setupController` are provided with access to `this.store`
- Possibility to define interfaces in your model and service files


# Limitations

## `this` context in computed properties:

The proper `this` context is not available inside computed property definitions yet:

```typescript
export default Ember.Object.extend({
  observedProp: "hello",

  computedProp: Ember.computed('observedProp', function() {
    let value = this.get('observedProp');
    //            ^-- type of `this` is unknown (any)
    //                so typechecking is disabled here
  })
}
```

## Potentially unsafe type for Ember.computed:

```typescript
let MyObj = Ember.Object.create({
  observedProp: "hello",

  computedProp: Ember.computed('observedProp', function() {
    let value : string = this.get('observedProp');
    // type of 'value' is declared to be string
    return value;
  })
}

let observedVal = MyObj.get('observedProp')
//      ^--- type of `observedVal` is string

let computedVal = MyObj.get('computedProp')
//      ^--- type of `computedVal` is string

MyObj.set('computedProp', 'a new value'):
// no warnings from typescript when overwriting a computed property
```

When writing code that uses MyObj, it will look like `computedProp` is a string, while it is actually a computed property.
Ideally we would be able to distinquish between readonly computed properties and simple properties of an object.
To workaround this problem, you could define your own interfaces for your models, and declare some properties to be `readonly`. 


## Advanced configuration

A little bit of configuration is needed to enable typechecking on models and services.
This section describes how to make use of the generated files in app/types/*.d.ts.

## ModelTypeIndex

EmberJS application code sometimes requires you to refer to your models, controllers or services by name (which is just a simple `string`). For example

```typescript
var user = this.store.createRecord('user', /* ... */);
// what is the type of `user`?

Ember.service.inject('my-logger-service');
// where to find 'my-logger-service'?
```

Typescript can't know that the string `"user"` maps to a `User` model. In order to link a models name to its type, we needed to provide Typescript with some sort of index:

```typescript
// pseudocode - please refer to the actual model index in app/types/application.d.ts
import User from "../models/user"

interface ModelTypeIndex {
  user: typeof User
}
```

Using the above `ModelTypeIndex`, Typescript is able to understand 

```typescript
var user = this.store.createRecord('user', /* ... */);
// user is of type Model

user.set('nonExistentKey', true);
//        ^^^^^^^^^^^^^^ this is a type error (key does not exist on user)

user.set('firstName', 42);
//                    ^^ this is a type error (int is not a string)
```


## ServiceTypeIndex

Service _names_ are mapped to their respective _types_ via the `ServiceTypeIndex`

Please refer to `ModelTypeIndex` for how this works, and the app/types/application.d.ts file.


----

# Reporting issues

If you come accross any issues, we would really appricate it if you
report it in our [Github Issue tracker](issues).



# Contributing

Pull requests are welcome! This addon still needs a lot of work, and we
are very happy to get input from the Ember and Typescript
communities.

