---
author: "Danny van der Biezen"
date: 2020-02-26
linktitle: How to overengineer a blog using Blazor WebAssembly - The architecture
title: How to overengineer a blog using Blazor WebAssembly - The architecture
---

Last week I decided I wanted to experiment a little bit with Blazor WebAssembly to get to know it a bit better. I have always wanted to have my own blog, but I was never sure if I had anything interesting to write about.  So my very first blog post will be about how I used Blazor to write a blog, which is quite nice :).

> The tech-savvy reader might have noticed that this blog is using [Hugo](https://gohugo.io/) instead of Blazor WebAssembly. A static website is a lot easier to maintain and host than a web app using a preview version of Blazor. I would recommend not to introduce unnecessary complexity into your applications unless you are experimenting like me ;).

## Tech stack
I decided on the following tech stack:
 - Database - MongoDB
 - Frontend - Blazor WebAssembly
 - Backend - ASP.NET Core
 - Realtime communication - ASP.NET Core SignalR
 
On January 28th, 2020, Microsoft released Blazor WebAssembly 3.2.0 Preview 1. With this release comes support for the .NET SignalR client. which can be used to communicate between the client and server using WebSockets. I haven't had to chance to use either of these so this seemed like a perfect way for me to learn a bit more about it. 

## Architecture
The idea behind this project is to make use of MongoDB Change Streams. Leveraging [Change Streams](https://www.mongodb.com/blog/post/an-introduction-to-change-streams) to stream back any changes to our blog posts from the database to our backend. Once we receive this data SignalR would then propagate this change to our frontend which would be updated in real-time. 
The Change Stream would be continuously listening for changes in the database, to achieve this I will be using a hosted service inside my ASP.NET Core server. 

A benefit to this approach is that my blog will be updated in real-time while only making changes to my database layer and leaving my deployed blog as is. I could then use any number of ways to update my blog using a different application like an android app, a web app or just inserting directly into the database like:
```JSON
    {
        "CreatedBy": "Admin",
        "CreatedOn": "2020-02-26T00:00:00.000+00:00",
        "Title": "My first blog post",
        "Description": "A short description",
        "Content": "My very first blog post"
    }
```
## Next up
Lots of text, not a lot of code. In the next part I will dive into the code and talk more about my experience using Blazor. Stay tuned!
