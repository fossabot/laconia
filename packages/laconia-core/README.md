# @laconia/core

[![CircleCI](https://circleci.com/gh/ceilfors/laconia/tree/master.svg?style=shield)](https://circleci.com/gh/ceilfors/laconia/tree/master)
[![Coverage Status](https://coveralls.io/repos/github/ceilfors/laconia/badge.svg?branch=master)](https://coveralls.io/github/ceilfors/laconia?branch=master)
[![Apache License](https://img.shields.io/badge/license-Apache-blue.svg)](LICENSE)

> 🛡️ Laconia Core — Micro dependency injection framework.

An AWS Lambda handler function is a single entry point for both injecting dependencies
and function execution. In non-serverless development, you can and will normally
only focus on the latter. This brings a unique challenge to AWS Lambda development
as it is very difficult to test a handler function when it is responsible for doing
both the object creations and the application run. @laconia/core is a simple dependency
injection framework for your Lambda code, hence solving this problem for you.

Laconia explicitly splits the responsibility of the object creations and Lambda function execution.
Laconia also provides a simple way for you to execute your Lambda function
so that your unit tests will not execute the code that instantiates your Lambda dependencies.

## FAQ

Check out [FAQ](https://github.com/ceilfors/laconia#faq)

## Features

* Dependency injection
* Caching for heavy instance creation
* Reduce boilerplate code
* Avoid common Lambda programming error

One of the problem in AWS Lambda is the `UnhandledPromiseRejectionWarning` problem where
[you can't throw an error out of your handler function](https://stackoverflow.com/questions/49894595/unhandled-promise-rejection-on-aws-lambda-with-async-await).
By simply using `laconia`, you will avoid this common problem.

## Install

```
npm install --save @laconia/core
```

## Dependency Injection

To fully understand how Laconia's Dependency Injection works, let's have a look into an
example below. _This is not a running code as there are a lot of code that have been trimmed down,
full example can be found in the acceptance test: [src](packages/laconia-acceptance-test/src/place-order.js)
and [unit test](packages/laconia-acceptance-test/test/place-order.spec.js)_.

Lambda handler code:

```js
// Objects creation, a function that returns an object
const instances = ({ env }) => ({
  orderRepository: new DynamoDbOrderRepository(env.ORDER_TABLE_NAME),
  idGenerator: new UuidIdGenerator()
});

// Handler function, which do not have any object instantiations
exports.app = async (event, { orderRepository, idGenerator }) => {
  // Instances made available via destructuring
  await orderRepository.save(order);
};

exports.handler = laconia(exports.app).register(instances);
```

Unit test code:

```js
const app = require("../src/place-order").app;

// Creates a mock Laconia context
beforeEach(() => {
  lc = {
    orderRepository: {
      save: jest.fn().mockReturnValue(Promise.resolve())
    }
  };
});

// Runs app function without worrying about the objects creation
it("should store order to order table", async () => {
  await app(event, lc);

  expect(lc.orderRepository.save).toBeCalledWith(
    expect.objectContaining(order)
  );
});
```

Note that as you have seen so far, Laconia is not aiming to become a comprehensive
DI framework hence the need of you handle the instantiation of all of the objects by yourself.
It should theoretically be possible to integrate Laconia to other more comprehensive
NodeJS DI framework but it has not been tested.

### LaconiaContext

Laconia provides a one stop location to get all of the information you need for your Lambda
function execution via `LaconiaContext`. In a nutshell, LaconiaContext is just an object that
contains all of those information by using object property keys, hence you can destructure it
to get just the information you need.

#### AWS Event and Context

When Laconia is adopted, the handler function signature will change slightly. Without Laconia,
your handler signature would be `event, context, callback`. With Laconia, your handler
signature would be `event, LaconiaContext`. The `context` are always available in LaconiaContext.
`callback` is not made available as this should not be necessary anymore when you are
using Node 8, just `return` the value that you want to return to the caller inside the handler function.

Example:

```js
laconia((event, { context }) => true);
```

#### Environment Variables

It is very common to set environment variables for your Lambda functions.
This is normally accessed via `process.env`. Unit testing a Lambda function that
uses `process.env` is awkward, as you have to modify the `process` global variable and remember
to remove your change so that it doesn't affect other test.

For better unit testability, LaconiaContext contains the environment variables
with key `env`.

Example:

```js
laconia((event, { env }) => true);
```

#### Cache

When `#register` is called, all of the instances returned by the function specified will be
cached by default and will expire every 5 minutes. It is therefore a good practice to offload
some of the heavy operations that don't change on every Lambda run to `LaconiaContext`.
This feature can be turned off, see API section.

### API

#### `laconia(app)`

* `app(event, laconiaContext)`
  * This `Function` is called when your Lambda is invoked
  * Will be called with `laconiaContext` object, which can be destructured to retrieve your dependencies

Example:

```js
// Simple return value
laconia(() => "value");

// Return a promise and 'value' will be returned to the Lambda caller
laconia(() => Promise.resolve("value"));
```

#### `register(factory, options)`

Registers objects created by the factory function into LaconiaContext.
Objects registered here will be made available in the Lambda function execution.
You can pass an array for the list of array to be called in parallel.

* `factory(laconiaContext)`
  * This `Function` is called when your Lambda is invoked
  * When an `Array` is specified, the list of factories within the array will be called concurrently with Promise.all
  * An object which contains the instances to be registered must be returned
* `options`:
  * `cache`
    * `enabled = true`
      * Set to false to turn off caching
    * `maxAge = 300000`
      * Your factory will be called when the cache has reached its maximum age specified by this option

Example:

```js
// Register an object with key 'service'
laconia((event, { service }) => service.call()).register(() => ({
  service: new SomeService()
}));

// Register concurrent factories
const app = () => {};
laconia(app).register([
  ssmConfig.envVarInstances(),
  s3Config.envVarInstances()
]);

// Reduce maxAge
const app = () => {};
laconia(app).register(
  async () => ({
    /* heavy operations */
  }),
  {
    cache: {
      maxAge: 1000
    }
  }
);
```

#### `postProcessor(postProcessorFn)`

Upon Lambda runtime execution, every postProcessorFn will be called on every factory functions
individually.

* `postProcessorFn(instances)`
  * This `Function` is called when your Lambda is invoked

Example:

```js
// Print object registered
laconia((event, { service }) => service.call())
  .register(() => ({
    service: new SomeService()
  }))
  .postProcessor(instances => console.log(instances));

// Enable xray
const xray = require("@laconia/xray");

laconia((event, { service }) => service.call())
  .register(() => ({
    service: new SomeService()
  }))
  .postProcessor(xray.postProcessor());
```
