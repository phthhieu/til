# Microservice Introduction

__What makes microservice attractive__
It is _independence_:
* Freedom to pick the right tool
* Quick iteration
* Rewrites are a possibility
* Code quality and readability

---

__When designing our microservice-based architecture__
* Cross-cutting concerns: not deal with the detail
* Data sharing is hard
* Availability: need to be monitored to detect failures as early as possible
* Evolution: versioning services
* Automated deployment
* Interdependencies: Keep minimum
* Transport and data format: HTTP + JSON, AMQP,.etc

---

__Do things right__
* API proxying
* Logging
* Service discovery and registration
* Service dependencies
* Data sharing and synchronization
* Graceful failure
* Automated deployment and instantiation
