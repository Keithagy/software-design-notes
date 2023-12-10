# Designing Maintainable Frontends

_Or, how to write frontend code that doesn't rot._

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

In a maintainable frontend, developers can:

- Reason about the same concepts and behaviors as non-technical stakeholders.
- Change renders _only_ alongside evolving user stories
  instead of, say, a plumbing detail like API structure.
- Keep cognitive overhead low by maintaining application state [_parsimoniously_](https://www.oxfordreference.com/display/10.1093/oi/authority.20110803100346221).

This document shows you how to enable these goals by breaking up frontend code
into independently maintainable
**Render**, **Data** and **Foundation** layers.

It also includes some useful materials and heuristics for writing,
well, code that doesn't rot.

To start with a spoiler:

> TODO: Sequence diagram demonstrating
> foundation layer =>
> parsing from raw to domain =>
> data layer =>
> composition with other domains, computation of view if needed =>
> render layer =>
> computation of payload back to foundation domain if needed

## Code is Communication

> We write code for other humans.
> That machines can execute the code is almost beside the point.

That's why we learn frontend frameworks and use high-level languages like JavaScript!
If all we cared about were machine-executability, we'd be writing only assembly.

Hold yourself to a more interesting standard:

- INSTEAD OF:
  `Code compiles, makes the intended behavior possible.`
- STRIVE FOR:
  `Code intent leaps out at the reader, makes unintended behavior hard to introduce.`
  - Along the same lines, favor implementations / toolchains which
    [make invalid states unrepresentable](https://ybogomolov.me/making-illegal-states-unrepresentable).

As a programmer,
aim to _give your collaborators
**(which includes future you)**
a clear script of the idea_.

## Application Layering

Maintability is, at its core, about keeping change easy,
which goes hand in hand with keeping complexity manageable,
which in turn goes hand in hand with keeping contexts tightly scoped.

The less you need your reader to remember
(while still having working code),
the better.

The principle of modeling your data for minimal cognitive overhead
is not unique to frontend development.

Realize that this does not mean to write less code,
or to play code golf with your collaborators;
Verbosity is often worth its weight in clarity.

Structure the idea of your application so that it "peels apart",
layer by layer, like an onion.

Implementing for frontend-specific concerns guides us to these specific layers.

### Render Layer

If we were to take a straight-line approach to frontend development,
the render layer would be the whole application.

#### Render Responsibilities

It has two responsibilities:

- Defining UI
  - Layouts
  - Styling
- State management
  - List of users that should be shown
  - Triggering of re-rendering upon completion of network query
    or prolonged inactivity
  - Real-time payload driven re-rendering

#### Render Organization

This layer should be organized to closely follow the end user's route tree.
Just as this layer would comprise the entire application in a simpler architecture,
the organization on this layer looks the most like what
you would expect in a typical frontend application.

Notes specific to React / TSX:

- `.tsx` files should be named in `CapitalCamelCase` and organized per route tree
- Common folder names:
  - `components`
  - `layouts`
  - `helpers`
    - e.g. for outputting the display string
      for a given domain object consistently across far-flung views)

#### Parsimonious State Management

In a line: reap immense maintainability benefits by
keeping your dependency tree as light as possible.

Since most frontends rely on a node-based diffing algorithm on a component tree,
placing your state containers as far down the tree as you can
gives you performance benefits for free, since the framework
would not have to compare dependencies to check for a re-render
in the first place.

> Note: Above point is highly dependent on your framework's render-lifecycle paradigm.
> In particular, the above point is unlikely to apply to signal-based frameworks
> like Astro / Preact / Svelte.

Another framework-specific pointer:
_You probably don't need Redux, which is good news!_

This means you probably can save the complexity cost a large additional dependency
simply by writing context providers as and when required.

Avoid using context providers to circumvent prop-drilling, because
that usually makes the dependencies of a particular component harder to track.

- If you are needing to send a prop many levels down along a component tree,
  stop and consider if you can actually be placing your
  state container further down the tree instead.
- State that should truly be considered app-wide state
  SHOULD be placed in context providers, because it's accurate data modelling:
  - `loggedInUser`
  - `theme`

In a line, the nesting of stateful variables
across your component tree should effectively guide your reader's attention
and help them make sense of the intended user journeys.

#### Keep Views Immutable and Uni-Directional

If a given type is supposed to a _view_, that means it should be derived
from a domain object AND used only for rendering UI.

The architecture should preserve uni-directional dataflow,
from the foundation to the render layer.

Related point:
State-mutation payloads should be calculated based on the
domain object backing the view, not the view itself.

### Data Layer

Data layer is where the transformations are defined, which bridge
raw data streamed from the backend into the stateful views composed in the UI.

#### Data Responsibilities

It has two responsibilities:

- Model the domain object
  - `Burger`s should have
    `meatType` which can be `MeatType.Chicken`, `MeatType.Beef` or `MeatType.Fish`,
    `cheeseSlices` which should be a positive integer,
    `orderer` which should be the UUID of a user, etc.
  - `Dog`s should have
    `neuteredDate` which should be a valid timestamp,
    `vaccinations` which should be a list of UUIDs of enumerated vaccines, etc.
- Define transformations to / from the domain object
  - Transformations to the domain object are called after
    network queries return a response
    - `parseBurgerFromRaw`
  - Transformsations from the domain object are called either
    to compose domain objects for views,
    or to build network mutation payloads
    - `parseAdoptionTransactionFromDogAndUser`

#### Data Organization

This layer is broken up into domains, which are essentially concepts
that stand on their own in non-technical discussions.

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

> Domains often "melt" into one another, since humans
> often define things relationally / circularly;
> user stories often compose multiple domains.

In addition, how a domain "understands itself" frequently diverges
from what another domain "expects" of it.

- For instance, it is usually meaningless to talk about a **transaction**
  without bringing up the **users** who had been counterparties to the transaction
  and the **asset** being exchanged.
- BUT, whereas a user might "understand itself" as a holder of
  a `userId`, a `userName`, an `emailAddress`, a `postalCode` and so on,
  a transaction might only "expect" a `userId` of a counterparty.

In other words, domains are often _conceptually_ coupled,
but _implementationally_ decoupled.
How do we model this?

##### The solution

> Use the type system to explain to your reader
> how pieces of data relate to each other.

Who should hold the richer type system for a piece of data?
For instance, since robots and issues both care about operation states,
who should more richy describe + export type definitions relating to operation states?

This is where the importance of modelling the domain thoughtfully becomes clear.
Every piece of data that warrants rich typing should have a "owner domain"
that reflects the way _non-technical stakeholders_ think about the data.
Ultimately, this practice serves the end-goal of **allowing developers reason
in the same terms as non-technical stakeholders and keep the code ready to
change with evolving business requirements**.

Core principle:
The "owner domain" models the data richly, and is responsible for validation
while "dependent domains" specify the primitive they expect to consume
and assume it pre-validated.

For instance:

- Suppose we decide that an ice cream shop's `transactions` domain
  should care about the `flavors` sold in the transaction
- However it is the `iceCream` domain that is responsible for modelling
  an `iceCream`'s flavors out of a finite set of possible flavors
- It would then fall to the `iceCream` domain
  to hold the rich type system and validation of `flavor`s
- The `transactions` domain should take for granted that it gets some `flavor`,
  perhaps modelled just as the raw underlying `string`,
  that is guaranteed to be valid

##### Tools at your disposal

- Type aliases to imbue primitives with domain-specific context
- Exported domain objects that provide a clear target for the render layer
- Unexported interfaces as consumer contracts
- Nullable types that require explicit handling
- Enumerated types which enforce raw values to be one a finite valid set
- Exported `fromRaw` and `toDomain` / `toView` functions whic clearly
  demarcate interaction points for different application layers

### Foundation Layer

This is, simply, the plumbing layer.

#### Foundation Responsibilities

It has two respnsibilities:

- Defining base types that all data domains would rely on
  - `uuid`
  - `millisecondsSinceEpoch`
- Defining non-domain-specific, raw-data facing helper functions and APIs
  - `errorHandledParseDateTime`
  - `HTTPClientWithErrorDisplayConvention`,
    with associated `Get` / `Post` / `Put` / `Delete` methods
  - `WebSocketConnectionHandler` (**that does not model domain-specific payload handling**)

#### Foundation Organization

This layer should be organized as a collection of resources
to be accessed by the various data domains.

For instance, it might make sense that `uuid` and `millisecondsSinceEpoch`
live as type definitions in a `types` file,
while `errorHandledParseDateTime` is exported as a function from a `dateTime` file
and `HTTPClientWithErrorDisplayConvention` is a class exported from a `http` file.

## Stress-Free Development Through Layering

The next few subsections will illustrate this architectural approach in action
by responding to some common situations arising in frontend development.

### Writing frontends independently of the backend

What do you do when you need to put up a design, but your backend isn't ready yet?
You don't need your backend to start modelling your domain.
Your backend is simply a source of raw data that
then gets transformed into your domain objects,
and if you model it accordingly you get to mock your API calls
to any arbitrary level of complexity.

Once you have your domain modelled,
you can start writing your render flows based on it,
And since you only need knowledge of the user stories and
business logic to begin modelling your domain,
Only the work of transforming the backend's raw objects
(in whatever shape and number they arrive in)
into your application's domain objects should be blocked.

### Absorbing a change in user story

> TODO: Change exclusively happens on the render layer

### Absorbing a change in domain knowledge

> TODO: Change exclusively happens on the data layer, UNLESS change in domain model
> impacts domain composition / view calculation happening on the render layer

### Absorbing an API change

> TODO: Change exclusively happens on the data layer
> in the raw-data facing portion of the dataflow definition

## References

Ideas here have been influenced to no small extent by Domain Driven Design principles,
though I have done my best not to require specific knowledge of DDD.
Still, further reading on the topic is available here:

- [The Big Blue Book](https://www.domainlanguage.com/ddd/blue-book/)
- [The Big Red Book](https://www.promyze.com/why-read-red-book-domain-driven-design/)

Domain driven design tends to be thought of as a backend thing,
but I think the component concepts have mapped across to
frontend development quite well, as there are ultimately rewards
in both situations to having "clean cable management" by
organizing code around context-bound dataflows. More reading:

- [Designing Data-Intensive Applications](https://www.amazon.sg/Designing-Data-Intensive-Applications-Reliable-Maintainable/dp/1449373321)
- [Data Oriented Frontends](https://dev.to/basal/data-oriented-frontend-development-1mk3)
- [Domain Driven Frontends](https://betterprogramming.pub/domain-driven-architecture-in-the-frontend-i-d27fb71b5cb0)
