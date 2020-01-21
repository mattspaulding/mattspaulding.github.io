---
layout: post
title: Let's Build a Hacker News Clone with Dotnet Core and Angular
image: /images/dotnet-core-angular.jpg
excerpt: YCombinator's infamous "Hacker News" has been a reliable source of news for the hacker community and, over time, has garnered the attention of hipsters and hustlers. Today we are building an app that displays the best stories from Hacker News.
---

![dotnet-core-angular]({{ site.baseurl }}/images/dotnet-core-angular.jpg)

YCombinator's infamous [Hacker News](https://news.ycombinator.com) has been a reliable source of news for the hacker community and, over time, has garnered the attention of hipsters and hustlers. 

Today we are building an app that displays the best stories from Hacker News.

>Take a look at the [demo](https://hacker-news-dotnet-angular.azurewebsites.net)

## Built with

* Dotnet Core v3.1
* Angular v8.2
* Angular Material
* Official Hacker News API => [https://github.com/HackerNews/API](https://github.com/HackerNews/API)

## About the Hacker News API

Before we get started, you should know that the Hacker News API is not great. Its a bit clunky to say the least. But don't take it from me. This is what the developers have to say about it from the official docs.

>The v0 API is essentially a dump of our in-memory data structures. We know, what works great locally in memory isn't so hot over the network. Many of the awkward things are just the way HN works internally. Want to know the total number of comments on an article? Traverse the tree and count. Want to know the children of an item? Load the item and get their IDs, then load them. The newest page? Starts at item maxid and walks backward, keeping only the top level stories. Same for Ask, Show, etc.

>I'm not saying this to defend it - It's not the ideal public API, but it's the one we could release in the time we had. While awkward, it's possible to implement most of HN using it.

While awkward, it is doable. And creates some unique coding situations to overcome.

Let's get started, shall we?

## Getting Started

### Prerequisites

Get the newest vscode => [https://code.visualstudio.com/download](https://code.visualstudio.com/download)

Get the newest LTS node => [https://nodejs.org](https://nodejs.org)

### Installation

1. Open a terminal where you want the project to be installed

1. Clone the repository

   ```sh
   git clone https://github.com/mattspaulding/hacker-news-dotnet-angular.git
   ```

1. Navigate into the repository

    ```sh
    cd hacker-news-dotnet-angular
    ```

1. Open the project in vscode

    ```sh
    code .
    ```

1. Click on the little debugger symbol on the far left nav then click the play button

    ![debug]({{site.baseurl}}/images/debug.png)

1. Wait about a minute. This first run needs to install some things. Then a browser should pop open on port 5001 and look something like this.

    >https://localhost:5001/

    ![start-success]({{site.baseurl}}/images/start-success.png)

## Project structure

That seemed a bit too easy. Let's see what is going on behind the curtain.

If you've ever built a SPA with an accompanying API you may have encountered the two-project structure. In such a setup you would create an API project and then create a separate SPA project. These two projects would run independently and be maintained, versioned, deployed independently. For example, you'd have to start the API project and then go into the SPA and `ng serve`.

That is not what is going on here.

This is a dotnet core project with an angular project living inside it. It is configured in such a way that we only need to press the `play` button and everything just gets going. Furthermore, maintenance, versioning, and deployment are a single project experience. I'll show you later how we can build the project and publish really easily to Azure. 

There are some scenarios where having the server and client seperated would be desired, but the one-project structure should work just fine in most cases.