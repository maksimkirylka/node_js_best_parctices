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

### 2.3 Distinguish operational vs programmer errors
Distinguishing the following two error types will minimize your app downtime and helps avoid crazy bugs: Operational errors refer to situations where you understand what happened and the impact of it – for example, a query to some HTTP service failed due to connection problem. On the other hand, programmer errors refer to cases where you have no idea why and sometimes where an error came from – it might be some code that tried to read an undefined value or DB connection pool that leaks memory. Operational errors are relatively easy to handle – usually logging the error is enough. Things become hairy when a programmer error pops up, the application might be in an inconsistent state and there’s nothing better you can do than to restart gracefully. 

Unless you really know what you are doing, you should perform a graceful restart of your service after receiving an “uncaughtException” exception event. Otherwise, you risk the state of your application, or that of 3rd party libraries to become inconsistent, leading to all kinds of crazy bugs.

### 2.4 Handle errors centrally, not within a middleware
