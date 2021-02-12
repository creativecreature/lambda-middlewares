# AWS Lambda middleware
This package helps you chain AWS lambda functions with an API that is similar to
Express middlewares.

The goal of this package is to help you and your team extract any common
functionality into new lambda middleware functions. You are then able to
order them however you want. The middlewares are executed left to right,
the last one will invoke your handler.

This package has no external dependencies, to avoid adding any bloat to your
lambda functions.

## Examples
Here are some examples for how this package could be used. The middlewares here
are *obviously* just for demo puroposes.


#### Supercharge your context with extra functionality(like a custom user object):

```js
import { withMiddlewares } from 'lambda-middleware'

const authenticate = async (token) => {
  const user = await authenticaitonService.getUser(token)
  return user
}

// Create a middleware for adding a custom user object to your lambda context.
const authMiddleware = async (event, context, next) => {
  const user = await authenticate()
  const userContext = { ...context, user }
  return await next(event, userContext)
}

// Create a middleware that relies on the user being there, and fetches additional information.
const orderMiddleware = async (event, context, next) => {
  const { user } = context
  const orders = await orderService.getOrders(user.email)
  const orderContext = { ...context, orders }
  return await next(event, orderContext)
}

// Your regular lambda handler.
const apiHandler = async (event, context) => {
  const { user, orders } = context
  // ... your specific logic

  return { statusCode: 200, body: JSON.stringify({ message: 'success' }) }
}

// Export your handler with middlewares.
export const handler = withMiddlewares([authMiddleware, orderMiddleware], apiHandler)
```


#### Create a reusable middleware for validating your requests:

```js
import { withMiddlewares } from 'lambda-middleware'

const orderRegex = new RegExp(/^[0-9A-F]{8}-[0-9A-F]{4}-4[0-9A-F]{3}-[89AB][0-9A-F]{3}-[0-9A-F]{12}$/i)

// Your custom middleware, takes in an additional next function, similar to express.
const validateOrderMiddleware = async (event, context, next) => {
  const orderId = event.pathParameters.orderId

  if !(orderRegex.test(orderId)) {
    // Validation failed, you can therefore exit early by returning and not invoking the *next* function.
    return { statusCode: 400, body: JSON.stringify(null) }
  }

  // Validation passed, invoke the handler!
  return await next(event, context)
}

// Your regular lambda handler.
const apiHandler = async (event, context) => {
  // At this point you know that the orderId has been validated
  const orderId = event.pathParameters.orderId
  // ... your specific logic

  return { statusCode: 200, body: JSON.stringify({ message: 'success' }) }
}

// Export your handler with middlewares.
export const handler = withMiddlewares([validateOrderMiddleware], apiHandler)
```


#### Your middlewares can wrap the handler, or other middlewares, to do cleanup

```js
import { withMiddlewares } from 'lambda-middleware'

// Your custom middleware, takes in an additional next function, similar to express.
const cleanupMiddleware = async (event, context, next) => {
  try {
    const res = await next(event, context)
    return res
  } catch (e) {
    console.log()
    return { statusCode: 500, body: { message: 'e.message' } }
  } finally {
    // Some code to cleanup
  }
}

// Your regular lambda handler.
const apiHandler = async (event, context) => {
  const result = await someService(event)
  return { statusCode: 200, body: JSON.stringify({ message: 'success' }) }
}

// Export your handler with middlewares.
export const handler = withMiddlewares([validateOrderMiddleware], apiHandler)
```


## Typescript
For examples in *typescript*, check out the test file in this project. There
you will find some examples on how to type your middleware functions.
