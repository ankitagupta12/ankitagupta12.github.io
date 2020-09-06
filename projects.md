---
layout: page
title: Projects
permalink: /projects
---

Some of the projects (mostly ruby gems) I have worked on are available publicly. These were largely inspired by problems I saw at work, that also seemed like more broadly applicable to other situations. So I ended up generalizing solutions to these problems where possible.


### [Brace-Comb (2019)](https://github.com/ankitagupta12/brace-comb)

This is a ruby library that makes it easier to build workflow management systems as a Directed Acyclic Graph. It provides a declarative format to define the dependencies between two actions in a workflow. For instance, in a delivery management system, the actions can be shopping and delivery of goods, and the dependencies between these actions is that shopping must be completed before delivery is attempted.

### [Kafka-Retryable (2018)](https://github.com/ankitagupta12/kafka-retryable)

Kafka-retryable builds application level handling of failures during consumption of Kafka messages. It allows specifying handlers when a Kafka consumer errors during message consumption. Failed messages are enqueued in a dead-letter queue, which can be configured as a topic in Kafka.

### [Pheromone (2018)](https://github.com/ankitagupta12/pheromone)

This is built for the use-case where any changes to data persisted via [ActiveRecord](https://guides.rubyonrails.org/active_record_basics.html) needs to be published as messages to Kafka. This is useful in building asynchronous, event-driven systems where updates to database objects trigger downstream consumers.

### [Dedup-Builds (2016)](https://github.com/ankitagupta12/dedup-builds)

Dedup-builds is a script that calls Travis CI and Drone APIs to cancel duplicate builds, except the build for the latest commit, for a particular pull request.

Fun fact: This was written when cancelling builds was not natively supported by these platforms. I _think_ by now both Travis and Drone have settings that allow you to enable cancelling duplicate builds.
