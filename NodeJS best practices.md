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

### 2.3 Distinguish operational vs programmer errors (Action Item!!!)
Distinguishing the following two error types will minimize your app downtime and helps avoid crazy bugs: Operational errors refer to situations where you understand what happened and the impact of it – for example, a query to some HTTP service failed due to connection problem. On the other hand, programmer errors refer to cases where you have no idea why and sometimes where an error came from – it might be some code that tried to read an undefined value or DB connection pool that leaks memory. Operational errors are relatively easy to handle – usually logging the error is enough. Things become hairy when a programmer error pops up, the application might be in an inconsistent state and there’s nothing better you can do than to restart gracefully. 

Unless you really know what you are doing, you should perform a graceful restart of your service after receiving an “uncaughtException” exception event. Otherwise, you risk the state of your application, or that of 3rd party libraries to become inconsistent, leading to all kinds of crazy bugs.

### 2.4 Handle errors centrally, not within a middleware (Action Item!!!)
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

### 2.10 Catch unhandled promise rejections (Action Item!!!)
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

### 2.12 Always await promises before returning to avoid a partial stacktrace (Action Item!!!)
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

## 3. Code Style Practices

### 3.1 Use ESLint
ESLint is the de-facto standard for checking possible code errors and fixing code style, not only to identify nitty-gritty spacing issues but also to detect serious code anti-patterns like developers throwing errors without classification. Though ESLint can automatically fix code styles, other tools like prettier and beautify are more powerful in formatting the fix and work in conjunction with ESLint.

### 3.2 Node.js specific plugins
TL;DR: On top of ESLint standard rules that cover vanilla JavaScript, add Node.js specific plugins like eslint-plugin-node, eslint-plugin-mocha and eslint-plugin-node-security

### 3.3 Start a Codeblock's Curly Braces on the Same Line
The opening curly braces of a code block should be on the same line as the opening statement:
```
// Do
function someFunction() {
  // code block
}
```
That's one of the pitfalls of JavaScript: automatic semicolon insertion. Lines that do not end with a semicolon, but could be the end of a statement, are automatically terminated.

### 3.4 Separate your statements properly
Use ESLint to gain awareness about separation concerns. Prettier or Standardjs can automatically resolve these issues.
You can use assignments and avoid using immediately invoked function expressions to prevent most of the unexpected errors.

### 3.5 Name your functions
Name all functions, including closures and callbacks. Avoid anonymous functions. This is especially useful when profiling a node app. Naming all functions will allow you to easily understand what you're looking at when checking a memory snapshot.

### 3.6 Use naming conventions for variables, constants, functions and classes
Use lowerCamelCase when naming constants, variables and functions, UpperCamelCase (capital first letter as well) when naming classes and UPPER_SNAKE_CASE when naming global or static variables. This will help you to easily distinguish between plain variables, functions, classes that require instantiation and variables declared at global module scope. Use descriptive names, but try to keep them short

```
// for global variables names we use the const/let keyword and UPPER_SNAKE_CASE
let MUTABLE_GLOBAL = "mutable value"
const GLOBAL_CONSTANT = "immutable value";
const CONFIG = {
  key: "value",
};

// examples of UPPER_SNAKE_CASE convention in nodejs/javascript ecosystem
// in javascript Math.PI module
const PI = 3.141592653589793;

// https://github.com/nodejs/node/blob/b9f36062d7b5c5039498e98d2f2c180dca2a7065/lib/internal/http2/core.js#L303
// in nodejs http2 module
const HTTP_STATUS_OK = 200;
const HTTP_STATUS_CREATED = 201;

// for class name we use UpperCamelCase
class SomeClassExample {
  // for static class properties we use UPPER_SNAKE_CASE
  static STATIC_PROPERTY = "value";
}

// for functions names we use lowerCamelCase
function doSomething() {
  // for scoped variable names we use the const/let keyword and lowerCamelCase
  const someConstExample = "immutable value";
  let someMutableExample = "mutable value";
}
```

### 3.7 Prefer const over let. Ditch the var
Using const means that once a variable is assigned, it cannot be reassigned. Preferring const will help you to not be tempted to use the same variable for different uses, and make your code clearer. If a variable needs to be reassigned, in a for loop, for example, use let to declare it. Another important aspect of let is that a variable declared using it is only available in the block scope in which it was defined. var is function scoped, not block-scoped, and shouldn't be used in ES6 now that you have const and let at your disposal.

### 3.8 Require modules first, not inside functions
Require modules at the beginning of each file, before and outside of any functions. This simple best practice will not only help you easily and quickly tell the dependencies of a file right at the top but also avoids a couple of potential problems.

### 3.9 Require modules by folders, as opposed to the files directly
When developing a module/library in a folder, place an index.js file that exposes the module's internals so every consumer will pass through it. This serves as an 'interface' to your module and eases future changes without breaking the contract.

### 3.10 Use the === operator
Prefer the strict equality operator === over the weaker abstract equality operator ==. == will compare two variables after converting them to a common type. There is no type conversion in ===, and both variables must be of the same type to be equal.

### 3.11 Use Async Await, avoid callbacks
Node 8 LTS now has full support for Async-await. This is a new way of dealing with asynchronous code which supersedes callbacks and promises. Async-await is non-blocking, and it makes asynchronous code look synchronous. The best gift you can give to your code is using async-await which provides a much more compact and familiar code syntax like try-catch

### 3.12 Use arrow function expressions (=>)
Though it's recommended to use async-await and avoid function parameters when dealing with older APIs that accept promises or callbacks - arrow functions make the code structure more compact and keep the lexical context of the root function (i.e. this).

## 4. Testing And Overall Quality Practices

### 4.1 At the very least, write API (component) testing
Most projects just don't have any automated testing due to short timetables or often the 'testing project' ran out of control and was abandoned. For that reason, prioritize and start with API testing which is the easiest way to write and provides more coverage than unit testing (you may even craft API tests without code using tools like Postman). Afterwards, should you have more resources and time, continue with advanced test types like unit testing, DB testing, performance testing, etc.

### 4.2 Include 3 parts in each test name (Action Item!!!)
Make the test speak at the requirements level so it's self-explanatory also to QA engineers and developers who are not familiar with the code internals. State in the test name what is being tested (unit under test), under what circumstances, and what is the expected result.

1. What is being tested? For example, the `ProductsService.addNewProduct` method
2. Under what circumstances and scenario? For example, no price is passed to the method
3. What is the expected result? For example, the new product is not approved

```
//1. unit under test
describe('Products Service', () => {
  describe('Add new product', () => {
    //2. scenario and 3. expectation
    it('should have pending approval status when no price is specified, ', () => {
      const newProduct = new ProductService().add(...);
      expect(newProduct.status).to.equal('pendingApproval');
    });
  });
});
```

### 4.3 Structure tests by the AAA pattern (Action Item => verify on this)
Structure your tests with 3 well-separated sections: Arrange, Act & Assert (AAA). The first part includes the test setup, then the execution of the unit under test, and finally the assertion phase. Following this structure guarantees that the reader spends no brain CPU on understanding the test plan.

### 4.4 Detect code issues with a linter
Use a code linter to check the basic quality and detect anti-patterns early. Run it before any test and add it as a pre-commit git-hook to minimize the time needed to review and correct any issue. Otherwise you may let pass some anti-pattern and possible vulnerable code to your production environment.

### 4.5 Avoid global test fixtures and seeds, add data per-test (Action Item!!!)
To prevent test coupling and easily reason about the test flow, each test should add and act on its own set of DB rows. Whenever a test needs to pull or assume the existence of some DB data - it must explicitly add that data and avoid mutating any other records.


### 4.6 Constantly inspect for vulnerable dependencies
Even the most reputable dependencies such as Express have known vulnerabilities. This can get easily tamed using community and commercial tools such as 🔗 npm audit and 🔗 snyk.io that can be invoked from your CI on every build.
Otherwise: Keeping your code clean from vulnerabilities without dedicated tools will require to constantly follow online publications about new threats. Quite tedious.

### 4.7 Tag your tests
 Different tests must run on different scenarios: quick smoke, IO-less, tests should run when a developer saves or commits a file, full end-to-end tests usually run when a new pull request is submitted, etc. This can be achieved by tagging tests with keywords like #cold #api #sanity so you can grep with your testing harness and invoke the desired subset. For example, this is how you would invoke only the sanity test group with Mocha: mocha --grep 'sanity'

Otherwise: Running all the tests, including tests that perform dozens of DB queries, any time a developer makes a small change can be extremely slow and keeps developers away from running tests

### 4.8 Check your test coverage, it helps to identify wrong test patterns (Action Item!!!)
Code coverage tools like Istanbul/NYC are great for 3 reasons: it comes for free (no effort is required to benefit this reports), it helps to identify a decrease in testing coverage, and last but not least it highlights testing mismatches: by looking at colored code coverage reports you may notice, for example, code areas that are never tested like catch clauses (meaning that tests only invoke the happy paths and not how the app behaves on errors). Set it to fail builds if the coverage falls under a certain threshold.

### 4.9 Inspect for outdated packages (Action Item!!!)
Use your preferred tool (e.g. npm outdated or npm-check-updates) to detect installed outdated packages, inject this check into your CI pipeline and even make a build fail in a severe scenario. For example, a severe scenario might be when an installed package is 5 patch commits behind (e.g. local version is 1.3.1 and repository version is 1.3.8) or it is tagged as deprecated by its author - kill the build and prevent deploying this version.
Otherwise: Your production will run packages that have been explicitly tagged by their author as risky.

### 4.10 Use production-like environment for e2e testing
End to end (e2e) testing which includes live data used to be the weakest link of the CI process as it depends on multiple heavy services like DB. Use an environment which is as close to your real production environment as possible.
Otherwise: Without docker-compose, teams must maintain a testing DB for each testing environment including developers' machines, keep all those DBs in sync so test results won't vary across environments.

### 4.11 Refactor regularly using static analysis tools (Action Item!!!)
Using static analysis tools helps by giving objective ways to improve code quality and keeps your code maintainable. You can add static analysis tools to your CI build to fail when it finds code smells. Its main selling points over plain linting are the ability to inspect quality in the context of multiple files (e.g. detect duplications), perform advanced analysis (e.g. code complexity), and follow the history and progress of code issues. Two examples of tools you can use are Sonarqube (2,600+ stars) and Code Climate (1,500+ stars).
Otherwise: With poor code quality, bugs and performance will always be an issue that no shiny new library or state of the art features can fix.

### 4.12 Carefully choose your CI platform (Jenkins vs CircleCI vs Travis vs Rest of the world)
Your continuous integration platform (CICD) will host all the quality tools (e.g. test, lint) so it should come with a vibrant ecosystem of plugins. Jenkins used to be the default for many projects as it has the biggest community along with a very powerful platform at the price of a complex setup that demands a steep learning curve. Nowadays, it has become much easier to set up a CI solution using SaaS tools like CircleCI and others. These tools allow crafting a flexible CI pipeline without the burden of managing the whole infrastructure. Eventually, it's a trade-off between robustness and speed - choose your side carefully.
Otherwise: Choosing some niche vendor might get you blocked once you need some advanced customization. On the other hand, going with Jenkins might burn precious time on infrastructure setup.

###  4.13 Test your middlewares in isolation (Action Item!!!)
When a middleware holds some immense logic that spans many requests, it is worth testing it in isolation without waking up the entire web framework. This can be easily achieved by stubbing and spying on the {req, res, next} objects.
Otherwise: A bug in Express middleware === a bug in all or most requests.

