# The Jot
For help thinking. 

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