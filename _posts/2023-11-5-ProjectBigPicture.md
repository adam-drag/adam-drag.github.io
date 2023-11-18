---
layout: post
title: A Big Picture of the System
---
Since this is more of a dev blog than a deep dive into logistics, I won't go into business logic much for now. When designing the system architecture, I had to ask myself a crucial question: Do I want to create a system to sell or do I want to have fun and learn as much as possible? The answer was clear – I chose the latter. Before you judge the architecture and my decisions, please understand that I acknowledge I'm overengineering it. I could achieve the same functionalities with a much simpler architecture and finish the project in less time.

The architecture is event-driven, though not purely so, where the system's state is persisted only in events and can be reconstructed by replaying them from scratch or a snapshot. I might aim for that in the future, but it would add too much complexity. It's a semi-event-driven approach, using events to have highly decoupled system elements, allowing different engineers to work on them without causing problems for each other. I'm using a mono repo for everything since I'm the only developer, making life easier. However, the code structure is designed to allow splitting the project into multiple small, component-focused repos.

Some high-level details: the entire system will be deployed in AWS, with event processing in Lambdas and events delivered through SNS. After spending some time deciding which database to use, considering the well-defined entity schema with lots of relationships between entities, I settled on RDS. I know scalability is a potential issue, but realistically, I won't have enough traffic for it to be a problem. If scalability becomes a concern, I'd consider DynamoDB due to its low maintenance cost and AWS's efficient scalability handling.

Thanks for reading, and have a great day!