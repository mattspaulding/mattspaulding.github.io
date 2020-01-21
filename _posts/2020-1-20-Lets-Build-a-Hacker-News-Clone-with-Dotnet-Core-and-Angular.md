---
layout: post
title: Let's Build a Hacker News Clone with Dotnet Core and Angular
image: /images/dotnet-core-angular.jpg
excerpt: Today we are building an app that displays the best stories from Hacker News.
---

![dotnet-core-angular]({{ site.baseurl }}/images/dotnet-core-angular.jpg)

Today we are building an app that displays the best stories from Hacker News.

>Take a look at the [demo](https://hacker-news-dotnet-angular.azurewebsites.net)

It is built with:

* Dotnet Core v3.1
* Angular v8.2
* Angular Material
* Official Hacker News API => [https://github.com/HackerNews/API](https://github.com/HackerNews/API)

## Prerequisites

Get newest vscode => [https://code.visualstudio.com/download](https://code.visualstudio.com/download)

Get the newest LTS node => [https://nodejs.org](https://nodejs.org)

## Getting Started

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