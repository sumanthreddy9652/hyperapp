How it works
The section below is not intended as a beginner's intro to hyperapp, but rather some notes with tradeoffs to consider when building a bigger application with Hyperapp.

Build
I use parcel to automate application build. Ideally I'd like to avoid bundling and transpilation during development as most modern browsers support the latest ES features. Unfortunately browsers can't resolve npm based dependencies with imports. Also, browsers don't understand JSX. My main motivation for using JSX over ES6 tagged template literals is better code formatting support in most tools.

index.html is our main entry point and parcel resolves graph of dependencies from there. First it goes to src/main.js and then resolves all the other JS dependencies.

App structure
main.js - injects browser specific dependencies to our app. It's only used for production code, in tests we can inject test doubles instead of real fetch or localStorage. In the remaining parts of my codebase I try to avoid using window or any other global object in favor of argument passing (dependency injection).

app.js - this is where we build main dependency graph for our application. Here we initialize localStorage based session repository and fetch based API gateway and inject them into corresponding action factories.

Actions (article, articles, auth, editor, profile, settings) are split along business capabilities. Actions manage their parts of the state. To keep actions testable we always inject all heavy dependencies. Scaling application logic and state follows a simple strategy: if a given object gets too big split it into smaller objects and delegate.

All view code resides in one view directory and consists only of view functions/page fragments. Scaling view code follows a simple strategy: if a given view fragment gets too big split it into smaller fragments and delegate.

Hyperapp itself expects state, actions, view and mount point. app function is our main contact point with the framework.

app(state, actions, view, document.body)
In app.js we provide initial value of the application state, then we aggregate actions from all the modules that make up the app and finally we provide aggregate view of the entire app.

Data model
State/model is inspired by Elm RealWorld. To keep it simple I don't use things like Maybe types as they don't seem to be idiomatic in the JS world. Data modelling is where I missed Elm like static type system the most.

Hyperapp enforces one big state object and doesn't allow you to mutate it. It raises 2 questions:

how to get deeply nested fields w/o duplicating code in many places
how to update deeply nested field in the immutable world
Usually people write selectors to solve the first problem.
To overcome the second issue we either use Object.assign() or spread objects extensively.

Lenses (e.g. provided by ramda) solve this problem is a more composable, realiable and elegant way.

Instead of writing

const authorSelector = state => state.page.article.author // read
{...state, page: {...state.page, article: {...state.page.article, author: 'new author'}}} //write
we can do the following

R.view(R.lensPath(['page', 'article', 'author']), state) //read
R.set(R.lensPath(['page', 'article', 'author']), 'new author', state) // simple write
R.over(R.lensPath(['page', 'article', 'author']), R.toUpper, state) // more complex write 
Please note that there's a symmetry between getting and setting the data.

For state derived values/selectors I decided to write some JS code on the top of lenses view without any extra library like reselect. I can add memoization on top when needed. However my runtime performance analysis didn't show bottlenecks in this area so far.

Views
All view related code is pure functions/view fragments/stateless components mapping state to Virtual DOM in JSX.

I reused most of the code from React-Redux RealWorld. The main difference is that there's only functions and no components with local state. I find it easier to reason about and manage one state object rather than state split between different components.

I decided to pass all function arguments explicitly at the expense of more boilerplate. Another option would be to use lazy components which is basically Hyperapp equivalent of React's mapStateToProps/mapDispatchToProps where we allow each view fragment to peek inside state and actions.

Compared to React conventions, hyperapp favors native DOM events and attributes (e.g onclick over onClick and class over className). This is nice since I can transfer my DOM knowledge.

Routing
Routing state is part of the main state object. Router also enhances actions with a history.pushState() wrapper (actions.location.go). I copy pasted parts of the original @hyperapp/router that I found useful and they live in the router directory (locations and parseRoute). I don't use Route components modelled after React Router components as they encourage data fetching from lifecycle hooks which I don't like. I also prefer regular anchor tags over Link components which don't allow to inject API calls before we change URL. I use internal-nav-helper to have a single click handler in the root element of the application. By default all links end up in this common handler unless you opt out by stopping event propagation.

Client side routing cycle looks like this:

click a link with href
load new page (set new page name based on href and preload its data from API)
dynamically load page by name (API data arrives in the background) This way our view functions don't need any oncreate/onupdate hooks (think componentDidMount and co. in the React world). We select views dynamically based on the current page name, not on the current state of the URL.
To make back button work correctly we also track popstate events.

API and async patterns
Most async code in this app revolves around API interactions that are simple enough to be handled with promises (no sagas/observables etc.). I factored out repeatable async flows into helper functions. E.g. actions/forms.js has withErrorHandling function that keeps track of starting/stoping async requests and handling potential errors. Future version of Hyperapp should have the concept of "effects as data" similar to Elm and I'm looking forward to it as it should make testing easier.

Error handling
I only did the minimum to replicate basic form error handling (client errors) as shown in the reference React version. We could go one step further and add better server error handling for API errors/network failures.
