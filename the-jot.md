#ideas #feed

A scratchpad for help thinking. 
# 2025 December

## 2025-11-23
### Page/UI specific endpoints

It may make sense to utilize discrete DDD-style endpoints only in so much that it's useful - discrete actions or and async read. 

I worry that composition of DDD-style endpoints in the frontend clients may get in the way of doing things simply in the frontend. Resource management can become pretty complex, it's enticing to push more of this to the server. If an API is pretty discrete and decoupled at an entity level, which is reasonable, it might be cumbersome to juggle or make latency requirements. If the API is incomplete or inconsistent or capability-driven, it might result in strange bespoke frontend patterns and API decisions that increase complexity of the project or force you to make decisions about the API interface quicker than its due time. It occurs to me that maybe giving a space specifically for more UI-favorable aggregates in the API could be helpful for building this more discrete capability-driven API at a slower rate, providing additional time to make more thoughtful updates to that API. And I theorize that if your page aggregates are broad and make semantic sense, you are building an API that any client can use for data needs. 

It should be stated that building out discrete capabilities is also good and this layer should not detract from the implementation of discrete capability-driven API. You need to count on them to serve a spectrum of dynamic behaviors. 

Maybe create a namespace under some code name for separating out the endpoints in experiment. 

I think you would want to place guardrails around the idea while trialing. I'm open to pretty strict regulations:
* Used exclusively for initial page load data requirements; if trial successful, could move towards more generalized aggregate retrieval
  * Measures of success: Improve page load times? Stability? Frontend resource management complexity?
* Initial data load during page load must be kept under some latency requirement; async patterns must take over after that threshold.
* Living technical specification in the repository. Consider separate file for each endpoint. Description, Operational Requirements, Aggregates involved, Link to API (OpenAPI? More?) Docs
* API Docs must be informative and helpful. Should be as good as a usage guide. They are the usage guide.

Other thoughts (to be tested/checked):
* Dynamicism of page is inverse to the latency requirements?
  * That is to say, maybe highly dynamic pages _should_ require less data to render whilst static pages _may_ use more

## 2025-12-6

It would be cool to create a markdown server. Like something I can embed in my webapp. I wonder how obsidian does it. 

## 2025-12-12
Happy Birthday, Mom. 

Want to keep things up to date and rolling? Try being relentless with a daily iterative process. Be nonchalant. Nobody need know you're doing it. 
* For mentorship, try a question a day, possibly for multiple people. Record the answers.
* To keep architecture, design, decisions, and rules up to date, engage people in sprintly decision reviews.

Some thoughts on UI:
* Consider need and possible colocation of use*Page.ts hooks
* Consider the meaning of a "Headless UI"
  * Consider the meaning of "Headless UI" in context of pages
    * Decision: Avoid this vague term and utilize common domain-driven jargon. 
* What's the role of a page?
  * Perhaps Controller in the NextJS sense? Take endpoint parameters and pass them directly to a connected feature-centric component composite.
    * What is "connected feature-centric component composite"? That would be a feature-rich component made up of arbitrarily many arbitrary components. Other feature-centric component composites should be passed in as instances or factory props to ensure that they can be orchestrated.
      * Is there a friendlier name? Business component? Domain Component? Domain Component makes sense - business logic of your UI. These are above the level of structural
* What's the role of a layout?
  * Layer? Application. 
  * DRYness? Definitely, at least for UI nesting.
    * What about when the layout needs data? Serverside rendering? Hoisting needed? 
      * If hoisting out of client side is needed, how do we make that seamless?
      * Or what if you just didn't? What if layout were just frames that required no domain data and they used providers to build an API to facilitate communication _up_ from children components or inform of layout-related information _down_ (e.g. maybe feature flags). Maybe use  use*Page.ts hooks to marry the domain-specific APIs to layout APIs.
  * Can I pass multiple props to a layout?
* Layers
  * Application
    * Layouts, DRY application framing
    * Layout Hooks, hooks that allow children components to access and manipulate Layout state. Compose layout hooks down the Component Tree
    * Pages, akin to NestJS Controllers
    * Page Hooks, aggregate domain hooks and layout hooks, parameterized over SDK
  * Domain
    * Domain Components, business ui comprised
    * Domain Services, 
  * Infra
    * Domain Hooks, hooks that allow children components to access and manipulate Layout state. A React wrapper of domain-specific SDK
      * This may be the wrong layer. This might belong in the application layer. 
    * SDK - 
  * Presentation
    * Pure Components
    * Composite Components
* Consider the need for a *.server|client.ts convention, if applicable

## 2025-12-14
#zotero #obsidian
Found an application called Zotero today that could be useful. It's a source materials manager. It also fields RSS feeds. I could see it being useful for a couple reasons:
- Be up to date on work-related feeds. Save impactful articles to Zotero library and create notes, related to the Zotero library

### 2025-12-15
- [[01-tasks/Dishes]]