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
