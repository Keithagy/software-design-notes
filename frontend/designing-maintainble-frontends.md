# Designing Maintainable Frontends

_Or how to write front-end code that doesn’t rot._

<!--toc:start-->

- [Overview](#overview)
- [Code is Communication](#code-is-communication)
- [Application Layering](#application-layering)
  - [Render Layer](#render-layer)
    - [Render Responsibilities](#render-responsibilities)
    - [Render Organization](#render-organization)
    - [Parsimonious State Management](#parsimonious-state-management)
    - [Keep Views Immutable and Uni-Directional](#keep-views-immutable-and-uni-directional)
  - [Data Layer](#data-layer)
    - [Data Responsibilities](#data-responsibilities)
    - [Data Organization](#data-organization)
    - [Providing a Rich Model of Data-Domain Relationships via Type Systems](#providing-a-rich-model-of-data-domain-relationships-via-type-systems)
      - [The problem](#the-problem)
      - [The solution](#the-solution)
      - [Tools at your disposal](#tools-at-your-disposal)
  - [Foundation Layer](#foundation-layer)
    - [Foundation Responsibilities](#foundation-responsibilities)
    - [Foundation Organization](#foundation-organization)
- [Stress-Free Development Through Layering](#stress-free-development-through-layering)
  - [Writing frontends independently of the backend](#writing-frontends-independently-of-the-backend)
  - [Absorbing a change in user story](#absorbing-a-change-in-user-story)
  - [Absorbing a change in domain knowledge](#absorbing-a-change-in-domain-knowledge)
  - [Absorbing an API change](#absorbing-an-api-change)
- [References](#references)
<!--toc:end-->

## Overview

In a maintainable front-end, developers can:

- Reason about the same concepts and behaviors as non-technical stakeholders.
- Change renders _only_ alongside evolving user stories instead of, for instance, a plumbing detail like API structure.
- Keep cognitive overhead low by maintaining application state [_parsimoniously_](https://www.oxfordreference.com/display/10.1093/oi/authority.20110803100346221 "https://www.oxfordreference.com/display/10.1093/oi/authority.20110803100346221").

This document shows you how to enable these goals by breaking up front-end code into independently maintainable **Render**, **Data,** and **Foundation** layers.

It also includes some useful materials and heuristics for writing, well, code that doesn’t rot.

To start with a spoiler:

> TODO: Sequence diagram demonstrating foundation layer => parsing from raw to domain => data layer => composition with other domains, computation of view if needed => render layer => computation of payload back to foundation domain if needed

## Code is Communication

> We write code for other humans. That machines can execute the code is almost beside the point.

That’s why we learn front-end frameworks and use high-level languages like JavaScript! If all we cared about were machine-executability, we’d be writing only assembly.

Hold yourself to a more interesting standard:

- INSTEAD OF: `Write code that compiles and makes the intended behavior possible.`
- STRIVE FOR: `Write code that conveys intent clearly to the reader and makes unintended behavior hard to introduce.`
    - Along the same lines, favor implementations/toolchains which [make invalid states unrepresentable](https://ybogomolov.me/making-illegal-states-unrepresentable "https://ybogomolov.me/making-illegal-states-unrepresentable").

As a programmer, aim to _give your collaborators **(including future you)** a clear script for the idea._
## Application Layering

Maintainability is, at its core, about keeping change easy, which goes hand in hand with conscientiously managing complexity, which in turn goes hand in hand with keeping contexts tightly scoped.

The less you need your reader to remember (while still having working code), the better.

The principle of modeling your data for minimal cognitive overhead is not unique to front-end development.

Please realize that this does not mean writing less code or playing code golf with your collaborators; Verbosity is often worth its weight in clarity.

Structure the idea of your application so that it “peels apart” layer by layer, like an onion.

Implementing toward frontend-specific concerns leads us to the following specific application layers.

### Render Layer

The render layer would be the whole application if we take a straight-line approach to front-end development.

#### Render Responsibilities

It has two responsibilities:

- Defining UI
    - Layouts
    - Styling
- State management
    - List of users that should be shown
    - Triggering of re-rendering upon completion of network query or prolonged inactivity
    - Real-time payload-driven re-rendering
#### Render Organization

This layer should be organized to follow the end user’s route tree closely. Just as this layer would comprise the entire application in a simpler architecture, the organization on this layer looks the most like what you would expect in a typical front-end application.

Notes specific to React / TSX:

- `.tsx` files should be named in `CapitalCamelCase` and organized per route tree
- Common folder names:
    - `components`
    - `layouts`
    - `helpers`,
        - e.g., for outputting the display string for a given domain object consistently across far-flung views)
#### Parsimonious State Management

In a line: reap immense maintainability benefits by keeping your dependency tree as light as possible.

Since most front-ends rely on a node-based diffing algorithm on a component tree, placing your state containers as far down the tree as possible gives you free performance benefits since the framework would not have to compare dependencies to check for a re-render in the first place.

> Note: Above point is highly dependent on your framework’s render-lifecycle paradigm. In particular, the above point is unlikely to apply to signal-based frameworks like Astro / Preact / Svelte.

Another framework-specific pointer: _You probably don’t need Redux, which is good news!_

This means you can probably save the complexity cost of a significant additional dependency simply by writing context providers as and when required.

Avoid using context providers to circumvent prop-drilling because that usually makes the dependencies of a particular component harder to track.

- If you need to send a prop many levels down along a component tree, stop and consider if you can place your state container further down the tree instead.
- State that should genuinely be considered app-wide state SHOULD be placed in context providers because it’s accurate data modeling:
    - `loggedInUser`
    - `theme`

In a line, the nesting of stateful variables across your component tree should effectively guide your reader’s attention and help them make sense of the intended user journeys.

#### Keep Views Immutable and Uni-Directional

If a given type is supposed to be a _view_, it should be derived from a domain object AND used only for rendering UI.

The architecture should preserve uni-directional data flow from the foundation to the render layer.

Related point: State-mutation payloads should be calculated based on the domain object backing the view, not the view itself.

### Data Layer

The data layer is where the transformations are defined, which bridge raw data streamed from the backend into the stateful views composed in the UI.

#### Data Responsibilities

It has two responsibilities:

- Model the domain object
    - `Burger`s should have `meatType` which can be `MeatType.Chicken`, `MeatType.Beef` or `MeatType.Fish`, `cheeseSlices` which should be a positive integer, and `orderer`, which should be the UUID of a user, etc.
    - `Dog`s should have a `neuteredDate,` a valid timestamp, `vaccinations,` a list of UUIDs of enumerated vaccines; etc.
- Define transformations to/from the domain object
    - Transformations to the domain object are called after network queries return a response
        - `parseBurgerFromRaw`
    - Transformations from the domain object are called either to compose domain objects for views or to build network mutation payloads
        - `parseAdoptionTransactionFromDogAndUser`
#### Data Organization

This layer is broken into domains, essentially concepts that stand independently in non-technical discussions.

Examples of what domain objects might look like:

- For a pet store:
    - customers
    - transactions
    - pets
    - land animals
    - ownership licenses
    - marine animals, etc.
- For a keyboard layout configuration tool:
    - layouts (e.g. Qwerty, Dvorak, Colemak)
    - bindings
    - layers
    - keyboard models, etc.
- For a robotics company:
    - robot models
    - operation states
    - battery levels
    - issue notifications, etc.
#### Providing a Rich Model of Data-Domain Relationships via Type Systems

##### The problem

> Domains often “melt” into one another, since humans often define things relationally / circularly; user stories often compose multiple domains.

In addition, how a domain “understands itself” frequently diverges from what another domain “expects” of it.

- For instance, it is usually meaningless to talk about a **transaction** without bringing up the **users** who had been counterparties to the transaction and the **asset** being exchanged.
- BUT, whereas a user might “understand itself” as a holder of a `userId`, a `userName`, an `emailAddress`, a `postalCode`, and so on, a transaction might only “expect” a `userId` of a counterparty. 

In other words, domains are often _conceptually_ coupled but _implementationally_ decoupled. How do we model this?

##### The solution

> Use the type system to explain to your reader how pieces of data relate to each other.

Who should hold the richer type system for a piece of data? For instance, since robots and issues both care about operation states, which should more richly describe + export type definitions relating to operation states?

This is where the importance of modeling the domain thoughtfully becomes clear. Every piece of data that warrants rich typing should have an “owner domain” reflecting how _non-technical stakeholders_ think about the data. Ultimately, this practice allows **developers to reason in the same terms as non-technical stakeholders and keep the code ready to change with evolving business requirements**.

Core principle: The “owner domain” models the data richly and is responsible for validation, while “dependent domains” specify the primitive they expect to consume and assume it pre-validated.

For instance:

- Suppose we decide that an ice cream shop’s `transactions` domain should care about the `flavors` sold in the transaction
- However, the iceCream domain is responsible for modeling an `iceCream`’s flavors out of a finite set of possible flavors.
- It would then fall to the `iceCream` domain to hold the rich type system and validation of `flavor`s.
- The `transactions` domain should take for granted that it gets some `flavor`, perhaps modeled just as the raw underlying `string`, that is guaranteed to be valid.    

##### Tools at your disposal

- Type aliases to imbue primitives with domain-specific context
- Exported domain objects that provide a clear target for the render layer
- Unexported interfaces as consumer contracts
- Nullable types that require explicit handling
- Enumerated types that enforce raw values to be one a finite valid set
- Exported `fromRaw` and `toDomain` / `toView` functions, which demarcate interaction points for different application layers

### Foundation Layer

This is the plumbing layer.

#### Foundation Responsibilities

It has two responsibilities:

- Defining base types that all data domains would rely on    
    - `uuid`
    - `millisecondsSinceEpoch`
- Defining non-domain-specific, raw-data-facing helper functions and APIs
    - `errorHandledParseDateTime`
    - `HTTPClientWithErrorDisplayConvention`, with associated `Get` / `Post` / `Put` / `Delete` methods
    - `WebSocketConnectionHandler` (**that does not model domain-specific payload handling**)

#### Foundation Organization

This layer should be organized as a collection of resources to be accessed by the various data domains.

For instance, it might make sense that `uuid` and `millisecondsSinceEpoch` live as type definitions in a `types` file, while `errorHandledParseDateTime` is exported as a function from a `dateTime` file and `HTTPClientWithErrorDisplayConvention` is a class exported from an `http` file.

## Stress-Free Development Through Layering

The following subsections will illustrate this architectural approach by responding to some common situations arising in front-end development.

### Writing front-ends independently of the backend

What do you do when you need to put up a design but your backend isn’t ready yet? You don’t need your backend to start modeling your domain. Your backend is simply a source of raw data that transforms into your domain objects. If you model it accordingly, you get to mock your API calls to any arbitrary level of complexity.

Once you have your domain modeled, you can start writing your render flows based on it. Since you only need knowledge of the user stories and business logic to begin modeling your domain, only the work of transforming the backend’s raw objects (in whatever shape and number they arrive in) into your application’s domain objects should be blocked.

### Absorbing a change in user story

> TODO: Change exclusively happens on the render layer

### Absorbing a change in domain knowledge

> TODO: Change exclusively happens on the data layer, UNLESS change in domain model impacts domain composition / view calculation happening on the render layer

### Absorbing an API change

> TODO: Change exclusively happens on the data layer in the raw-data facing portion of the dataflow definition

## References

Ideas here have been influenced to no small extent by domain-driven design(DDD) principles, though I have done my best not to require specific knowledge of DDD. Still, further reading on the topic is available here:

- [The Big Blue Book](https://www.domainlanguage.com/ddd/blue-book/ "https://www.domainlanguage.com/ddd/blue-book/")
- [The Big Red Book](https://www.promyze.com/why-read-red-book-domain-driven-design/ "https://www.promyze.com/why-read-red-book-domain-driven-design/")

Domain-driven design tends to be considered more relevant to the backend. Still, I think the component concepts have mapped across to front-end development quite well, as there are ultimately rewards in both situations to having “clean cable management” by organizing code around context-bound dataflows. More reading:

- [Designing Data-Intensive Applications](https://www.amazon.sg/Designing-Data-Intensive-Applications-Reliable-Maintainable/dp/1449373321 "https://www.amazon.sg/Designing-Data-Intensive-Applications-Reliable-Maintainable/dp/1449373321")
- [Data-Oriented Front-ends](https://dev.to/basal/data-oriented-frontend-development-1mk3 "https://dev.to/basal/data-oriented-frontend-development-1mk3")
- [Domain Driven Front-ends](https://betterprogramming.pub/domain-driven-architecture-in-the-frontend-i-d27fb71b5cb0 "https://betterprogramming.pub/domain-driven-architecture-in-the-frontend-i-d27fb71b5cb0")