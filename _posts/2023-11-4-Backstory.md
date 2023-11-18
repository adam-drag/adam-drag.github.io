---
layout: post
title: Starting My Private Project; Warehouse Inventory Management System
---

I decided to start creating a warehouse inventory management system because of my experience in supply chain management over six years. During that time, I was in charge of keeping track of inventory levels for hundreds of products in a factory that made spice mixtures, dealing with raw materials weighing over 100 tons each week. Despite the job being stressful, I found it enjoyable due to the challenges.

The main reason for this project was the limitations of the existing warehouse management system. It could only handle purchase orders and basic inventory tasks, lacking important features for making decisions, like forecasting and usage analysis. At 21 years old, with no programming experience, I felt the need to take action. My colleagues were printing lots of stock level pages and current production plans to plan new deliveries. This led to overstocking while essential raw materials were often insufficient.

Being good with Excel, I initially tried to improve inventory management using it instead of the existing system. I found I could pull reports in .csv format, including purchase orders, sales orders, and production plans. So, my own Excel-based stock management tools were born. Initially, it was a simple statistical sheet with basic math to get an overview of the current situation.

Realizing its limitations, I explored MRP (Material Requirement Planning) systems that focus on managing stock levels by constantly assessing demand versus current stock versus scheduled deliveries. I created a tool that connected these three values in an easy-to-read way, initially using only Excel functions. As this tool gained attention from my boss and colleagues, it evolved to meet their needs.

The initial challenge of managing multiple 'MRP files' for each user led to a significant change in the user interface. Excel cells and functions were no longer enough for the new features, so I implemented small pop-up windows using VBA (Visual Basic for Applications), supported by Excel. As I gained more experience with VBA, I tackled the issue of multiple files by moving user-manipulated data to an Access database. This decision was based on its ease of integration with VBA and my limited background in IT.

For about four years, this system served as the main tool for managing inventory levels in the entire factory, resulting in a significant reduction in stock value while improving the availability of raw materials. When I left the company, the stock value was close to $2 million, and they were transitioning to using SAP.

Now, armed with years of experience as a software engineer, I am revisiting a similar project, intending to develop it in a more professional way. Rather than thinking of it as something to sell, I see it as an opportunity to engage in something enjoyable and, most importantly, to learn as much as possible.