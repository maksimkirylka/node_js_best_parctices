# NodeJS best practices

## 1. Project Structure Practices

### 1.1 Structure your solution by components
Partition your code into components, each gets its folder or a dedicated codebase, and ensure that each unit is kept small and simple. The very least you should do is create basic borders between components, assign a folder in your project root for each business component and make it self-contained - other components are allowed to consume its functionality only through its public interface or API.

...if you were looking at the architecture of a library, you’d likely see a grand entrance, an area for check-in-out clerks, reading areas, small conference rooms, and gallery after gallery capable of holding bookshelves for all the books in the library. That architecture would scream: Library

### 1.2 Layer your components, keep the web layer within its boundaries
Each component should contain 'layers' - a dedicated object for the web, logic, and data access code. This not only draws a clean separation of concerns but also significantly eases mocking and testing the system.
Separate component code into layers: web, services, and Data Access Layer (DAL).

### 1.3 Wrap common utilities as npm packages
In a large app that constitutes a large codebase, cross-cutting-concern utilities like a logger, encryption and alike, should be wrapped by your code and exposed as private npm packages. This allows sharing them among multiple codebases and projects.

### 1.4 Separate Express 'app' and 'server'
Avoid the nasty habit of defining the entire Express app in a single huge file - separate your 'Express' definition to at least two files: the API declaration (app.js) and the networking concerns (WWW). For even better structure, locate your API declaration within components.

### 1.5 Use environment aware, secure and hierarchical config
 A perfect and flawless configuration setup should ensure
 - (a) keys can be read from file AND from environment variable
 - (b) secrets are kept outside committed code
 - (c) config is hierarchical for easier findability.
 There are a few packages that can help tick most of those boxes like rc, nconf, config, and convict.
 A reliable config solution must combine both configuration files + overrides from the process variables.
 
## 2. Error Handling Practices
 
### 2.1 Use Async-Await or promises for async error handling
Handling async errors in callback style is probably the fastest way to hell (a.k.a the pyramid of doom). The best gift you can give to your code is using a reputable promise library or async-await instead which enables a much more compact and familiar code syntax like try-catch.

### 2.2 Use only the built-in Error object
Whether you reject a promise, throw an exception or emit an error – using only the built-in Error object (or an object that extends the built-in Error object) will increase uniformity and prevent loss of information.

When raising the exception, it’s usually a good practice to fill it with additional contextual properties like the error name and the associated HTTP error code. To achieve this uniformity and practices, consider extending the Error object with additional properties, but be careful not to overdo it. It's generally a good idea to extend the built-in Error object only once with an AppError for all the application level errors, and pass any data you need to differentiate between different kinds of errors as arguments.

### 2.3 Distinguish operational vs programmer errors (Action Item)
Distinguishing the following two error types will minimize your app downtime and helps avoid crazy bugs: Operational errors refer to situations where you understand what happened and the impact of it – for example, a query to some HTTP service failed due to connection problem. On the other hand, programmer errors refer to cases where you have no idea why and sometimes where an error came from – it might be some code that tried to read an undefined value or DB connection pool that leaks memory. Operational errors are relatively easy to handle – usually logging the error is enough. Things become hairy when a programmer error pops up, the application might be in an inconsistent state and there’s nothing better you can do than to restart gracefully. 

Unless you really know what you are doing, you should perform a graceful restart of your service after receiving an “uncaughtException” exception event. Otherwise, you risk the state of your application, or that of 3rd party libraries to become inconsistent, leading to all kinds of crazy bugs.

### 2.4 Handle errors centrally, not within a middleware (Action Item)
Without one dedicated object for error handling, greater are the chances for inconsistent errors handling: Errors thrown within web requests might get handled differently from those raised during the startup phase and those raised by scheduled jobs. This might lead to some types of errors that are being mismanaged. This single error handler object is responsible for making the error visible, for example, by writing to a well-formatted logger, firing metrics using some monitoring product (like Prometheus, CloudWatch, DataDog, and Sentry) and to decide whether the process should crash. Most web frameworks provide an error catching middleware mechanism - A typical mistake is to place the error handling code within this middleware. By doing so, you won't be able to reuse the same handler for errors that are caught in different scenarios like scheduled jobs, message queue subscribers, and uncaught exceptions. Consequently, the error middleware should only catch errors and forward them to the handler.

### 2.5 Document API errors using Swagger or GraphQL
Let your API callers know which errors might come in return so they can handle these thoughtfully without crashing. For RESTful APIs, this is usually done with documentation frameworks like Swagger. If you're using GraphQL, you can utilize your schema and comments as well.

### 2.6 Exit the process gracefully when a stranger comes to town
When an unknown error occurs (a developer error, see best practice 2.3) - there is uncertainty about the application healthiness. Common practice suggests restarting the process carefully using a process management tool like Forever or PM2.

### 2.7 Use a mature logger to increase error visibility
A set of mature logging tools like Pino or Log4js, will speed-up error discovery and understanding. So forget about console.log.
 - Timestamp each log line. This one is pretty self-explanatory – you should be able to tell when each log entry occurred.
 - Logging format should be easily digestible by humans as well as machines.
 - Allows for multiple configurable destination streams. For example, you might be writing trace logs to one file but when an error is encountered, write to the same file, then into error file and send an email at the same time.

### 2.8 Test error flows using your favorite test framework
Whether professional automated QA or plain manual developer testing – Ensure that your code not only satisfies positive scenarios but also handles and returns the right errors. Testing frameworks like Mocha & Chai can handle this easily (see code examples within the "Gist popup")

### 2.9 Discover errors and downtime using APM products
Monitoring and performance products (a.k.a APM) proactively gauge your codebase or API so they can automagically highlight errors, crashes, and slow parts that you were missing

### 2.10 Catch unhandled promise rejections (Action Item)
Any exception thrown within a promise will get swallowed and discarded unless a developer didn’t forget to explicitly handle it. Even if your code is subscribed to process.uncaughtException! Overcome this by registering to the event `process.unhandledRejection` - this will ensure that any promise error, if not handled locally, will get its treatment:
```
process.on('unhandledRejection', (reason: string, p: Promise<any>) => {
  // I just caught an unhandled promise rejection,
  // since we already have fallback handler for unhandled errors (see below),
  // let throw and let him handle that
  throw reason;
});

process.on('uncaughtException', (error: Error) => {
  // I just received an error that was never handled, time to handle it and then decide whether a restart is needed
  errorManagement.handler.handleError(error);
  if (!errorManagement.handler.isTrustedError(error))
    process.exit(1);
});
```

### 2.11 Fail fast, validate arguments using a dedicated library
Assert API input to avoid nasty bugs that are much harder to track later.
We all know how checking arguments and failing fast is important to avoid hidden bugs (see anti-pattern code example below). If not, read about explicit programming and defensive programming

### 2.12 Always await promises before returning to avoid a partial stacktrace (Action Item)
Always do return await when returning a promise to benefit full error stacktrace. If a function returns a promise, that function must be declared as async function and explicitly await the promise before returning it.

```
async function throwAsync () {
  await null // need to await at least something to be truly async (see note #2)
  throw Error('with all frames present')
}

async function changedFromSyncToAsyncFn () {
  return await throwAsync()
}

async function asyncFn () {
  return await changedFromSyncToAsyncFn()
}

// 👍 now changedFromSyncToAsyncFn would present in the stacktrace
asyncFn().catch(console.log)
```
Log would be:
```
Error: with all frames present
    at throwAsync ([...])
    at changedFromSyncToAsyncFn ([...])
    at async asyncFn ([...])
```
Because only async functions may await, sync function would always be missed in async stacktrace if any async operation has been performed after the function has been called.
This is why resolving promises before returning them is the best practice for Node.js and not for the EcmaScript in general.
