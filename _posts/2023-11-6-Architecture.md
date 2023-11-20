---
layout: post
title: Architecture of the System
---

This post breaks down the raw architecture of "Stock-Mate," my inventory management system.

## Event-Driven Architecture

"Stock-Mate" is designed with an event-driven architecture, where the main components are loosely coupled and communicate using SNS (considering a potential move to SQS in a future post). Now, let's explore the initial system architecture, which is yet to be implemented.

[![Alt Text](/images/stock_mate_arch.png)](/images/stock_mate_arch.png)

## System Architecture Overview

The entire project will be hosted on AWS, utilizing EC2 instances in an AutoScaling group for the frontend – a Next.js web app. AWS Cognito will handle user authentication, allowing sign-ins using Google accounts. All traffic between the web app server and the backend will pass through API Gateway. The system will reside within a VPC hosting AWS RDS, and several Lambdas will be deployed.

### DataQueryLambda:

Responsible for pulling data from RDS, allowing users to retrieve products or purchase orders. It enforces a read-only policy, preventing inserts, updates, or deletions.

### EventEmitterLambda:

Exposes all insert/update/delete actions, validates received payloads, creates and persists events in RDS, and publishes them to the appropriate SNS topic.

### PersistenceServiceLambda:

Directly manipulates data in RDS and subscribes to events (e.g., NewProductScheduled) to persist changes in the database.

### IssueAnalyserLambda:

An intriguing component that reacts to product changes, analyzing stock levels, and making decisions on new purchase orders or order postponements. It interacts with ChatGPT for operator advice in case of detected issues.

### UsageAnalyserLambda:

Focuses on statistical analysis, examining historical product usage for seasonality, suggesting safety stock and maximum stock, and using machine learning algorithms for demand forecasting.

### BillingLambda:

Responsible for updating Stripe based on customer usage. Stripe, a developer-friendly payment gateway, is integrated to manage secure and customizable online payments.

That's more or less all. Thanks for reading.
