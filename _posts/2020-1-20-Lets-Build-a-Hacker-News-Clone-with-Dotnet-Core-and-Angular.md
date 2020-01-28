---
layout: post
title: Let's Build a Hacker News Clone with Dotnet Core and Angular
image: /images/dotnet-core-angular.jpg
excerpt: YCombinator's infamous "Hacker News" has been a reliable source of news for the hacker community and, over time, has garnered the attention of hipsters and hustlers. Today we are building an app that displays the best stories from Hacker News.
---

![dotnet-core-angular]({{ site.baseurl }}/images/dotnet-core-angular.jpg)

YCombinator's infamous [Hacker News](https://news.ycombinator.com) has been a reliable source of news for the hacker community and, over time, has garnered the attention of hipsters and hustlers.

Today we are building an app that displays the best stories from Hacker News.

> Take a look at the [demo](https://hacker-news-dotnet-angular.azurewebsites.net)

> See the final project in the [repository](https://github.com/mattspaulding/hacker-news-dotnet-angular)

## Built with

- Dotnet Core v3.1
- Angular v8.2
- Angular Material
- Official Hacker News API => [https://github.com/HackerNews/API](https://github.com/HackerNews/API)

## About the Hacker News API

Before we get started, you should know that the Hacker News API is not great. Its a bit clunky to say the least. But don't take it from me. This is what the developers have to say about it from the official docs.

> The v0 API is essentially a dump of our in-memory data structures. We know, what works great locally in memory isn't so hot over the network. Many of the awkward things are just the way HN works internally. Want to know the total number of comments on an article? Traverse the tree and count. Want to know the children of an item? Load the item and get their IDs, then load them. The newest page? Starts at item maxid and walks backward, keeping only the top level stories. Same for Ask, Show, etc.

> I'm not saying this to defend it - It's not the ideal public API, but it's the one we could release in the time we had. While awkward, it's possible to implement most of HN using it.

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

   > https://localhost:5001/

   ![start-success]({{site.baseurl}}/images/start-success.png)

## Project structure

That seemed a bit too easy. Let's see what is going on behind the curtain.

If you've ever built a SPA with an accompanying API you may have encountered the two-project structure. In such a setup you would create an API project and then create a separate SPA project. These two projects would run independently and be maintained, versioned, deployed independently. For example, you'd have to start the API project and then go into the SPA and `ng serve`.

That is not what is going on here.

This is a dotnet core project with an angular project living inside it. It is configured in such a way that we only need to press the `play` button and everything just gets going. Furthermore, maintenance, versioning, and deployment are a single project experience. I'll show you later how we can build the project and publish really easily to Azure.

Note: There are some scenarios where having the server and client seperated would be desired, but the one-project structure should work just fine in most cases.

## The Code

As previously mentioned, the project is structured as a dotnet project with an Angular project nested inside. First, let's look at the dotnet code.

### The Dotnet Code

The project is pretty simple and consists of a `HackerNewsController` and a `HackerNewsRepository`. First let's look at the repository.

#### Repository

The repository is responsible for interacting with the Hacker News API. As I said before, the Hacker News API is a bit strange. I will try to explain it.

First there is the 'BestStories' call.

> https://hacker-news.firebaseio.com/v0/beststories.json

This will return a response of an array of IDs. These IDs represent the stories.

```json
[
  22124489,
  22124929,
  22159385,
  22137279,
  22153304,
  ...
]
```

Next is the 'Item' call.

> https://hacker-news.firebaseio.com/v0/item/{0}.json

This returns the details of one of the IDs from the list.

```json
{
  "by": "clouddrover",
  "descendants": 360,
  "id": 22124489,
  "score": 1354,
  "time": 1579749137,
  "title": "Procrastination is about managing emotions, not time",
  "type": "story",
  "url": "https://www.bbc.com/worklife/article/20200121-why-procrastination-is-about-managing-emotions-not-time"
}

```

So what this means is that if you want the details of every item in the list, you need to make a seperate call for each item. This is not very efficient, but we will deal with that later.

The repository simply makes HTTP requests for this data.

```js
// HackerNewsRepository.cs

using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using System.Net.Http;
using hacker_news_dotnet_angular.Core.Interfaces;

namespace hacker_news_dotnet_angular.Infrastructure
{
    public class HackerNewsRepository : IHackerNewsRepository
    {

        private static HttpClient client = new HttpClient();

        public HackerNewsRepository()
        {

        }

        public async Task<HttpResponseMessage> BestStoriesAsync()
        {
            return await client.GetAsync("https://hacker-news.firebaseio.com/v0/beststories.json");
        }

        public async Task<HttpResponseMessage> GetStoryByIdAsync(int id)
        {
            return await client.GetAsync(string.Format("https://hacker-news.firebaseio.com/v0/item/{0}.json", id));
        }
    }
}
```

#### Controller

The controller is the entry point for the API and constructs the data retrieved by Hacker News into a format that will be consumed by the Angular client.

First get the best stories from the repository. And optionaly filter by a search term.

Once we have the list of best story IDs, we asyncronously call `GetStoryAsync` for each ID to get the details.

``` cs
// HackerNewsController.cs

using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Newtonsoft.Json;
using Microsoft.Extensions.Caching.Memory;
using hacker_news_dotnet_angular.Core.Interfaces;

namespace hacker_news_dotnet_angular.Controllers
{
    [ApiController]
    [Route("[controller]")]
    public class HackerNewsController : ControllerBase
    {
        private IMemoryCache _cache;

        private readonly IHackerNewsRepository _repo;

        public HackerNewsController(IMemoryCache cache, IHackerNewsRepository repository)
        {
            this._cache = cache;
            this._repo = repository;
        }

        public async Task<List<HackerNewsStory>> Index(string searchTerm)
        {
            List<HackerNewsStory> stories = new List<HackerNewsStory>();

            var response = await _repo.BestStoriesAsync();
            if (response.IsSuccessStatusCode)
            {
                var storiesResponse = response.Content.ReadAsStringAsync().Result;
                var bestIds = JsonConvert.DeserializeObject<List<int>>(storiesResponse);

                var tasks = bestIds.Select(GetStoryAsync);
                stories = (await Task.WhenAll(tasks)).ToList();

                if (!String.IsNullOrEmpty(searchTerm))
                {
                    var search = searchTerm.ToLower();
                    stories = stories.Where(s =>
                                       s.Title.ToLower().IndexOf(search) > -1 || s.By.ToLower().IndexOf(search) > -1)
                                       .ToList();
                }
            }
            return stories;
        }

        private async Task<HackerNewsStory> GetStoryAsync(int storyId)
        {
            return await _cache.GetOrCreateAsync<HackerNewsStory>(storyId,
                async cacheEntry =>
                {
                    HackerNewsStory story = new HackerNewsStory();

                    var response = await _repo.GetStoryByIdAsync(storyId);
                    if (response.IsSuccessStatusCode)
                    {
                        var storyResponse = response.Content.ReadAsStringAsync().Result;
                        story = JsonConvert.DeserializeObject<HackerNewsStory>(storyResponse);
                    }

                    return story;
                });
        }
    }
}
```

One interesting point here is the `_cache.GetOrCreateAsync` function. Since the Hacker News API isn't efficient, we cache all of the stories. For each subsequent request for a story, the user will recieve the cached version.

### The Angular Code

On construction of the home page, we make a call to our API to get the best Hacker News stories. When the results come back, the stories are displayed in a list of Angular Material cards. We have an input to search for keywords and when the story card is clicked, a new tab opens to the story.

```html
<!-- home.component.html -->

<h1>Hacker News Top Stories</h1>

<p *ngIf="!hackerNewsStories"><em>Loading...</em></p>
<div *ngIf="hackerNewsStories">
  <form>
    <mat-form-field>
      <input matInput placeholder="Search" (keyup)="search($event)">
    </mat-form-field>
  </form>
  <mat-card *ngFor="let story of hackerNewsStories" (click)="open(story.url)">
    <mat-card-content>
      <mat-card-title>{{ story.title }}</mat-card-title>
      <mat-card-subtitle>by {{ story.by }}</mat-card-subtitle>
    </mat-card-content>
  </mat-card>
</div>
```

```js
// home.component.ts

import { Component, Inject } from "@angular/core";
import { HttpClient } from "@angular/common/http";

@Component({
  selector: "app-home",
  templateUrl: "./home.component.html",
  styleUrls: ["./home.component.scss"]
})
export class HomeComponent {
  public hackerNewsStories: HackerNewsStory[];

  constructor(
    private http: HttpClient,
    @Inject("BASE_URL") private baseUrl: string
  ) {
    this.get("");
  }

  get(searchTerm: string) {
    this.http
      .get<HackerNewsStory[]>(
        `${this.baseUrl}hackernews?searchTerm=${searchTerm}`
      )
      .subscribe(
        result => {
          this.hackerNewsStories = result;
        },
        error => console.error(error)
      );
  }

  search(event: KeyboardEvent) {
    this.get((event.target as HTMLTextAreaElement).value);
  }

  open(url: string) {
    window.open(url, "_blank");
  }
}

interface HackerNewsStory {
  title: string;
  by: string;
  url: string;
}
```

## Publishing

Now that our project is working, let's publish it.

### Prerequisites

- An Azure subscription
  - You can start a free subscription if you do not already have one
  - [https://azure.microsoft.com](https://azure.microsoft.com)
- Azure App Service - extention for vscode
  - [azuretools.vscode-azureappservice](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azureappservice)

### Generate the deployment package

Create a `Release` package and put it in the `publish` directory

```sh
dotnet publish -c Release -o ./publish
```

You will notice the new `publish` directory.

![publish]({{site.baseurl}}/images/publish.png)

This contains your project optimized for production. Look closely and you will see that your angular project has also been built with this nice simple command.

![publish-angular]({{site.baseurl}}/images/publish-angular.png)

### Send it to Azure

1. Right click the `publish` folder and select `Deploy to Web App...`

   ![deploy-web-app]({{site.baseurl}}/images/deploy-web-app.png)

1. Select the subscription you want to create the Web App

1. Select `Create new Web App`

   ![new-web-app]({{site.baseurl}}/images/new-web-app.png)

1. Enter a name for the Web App

And that's it. Your project is now live in production.

This is my live URL: [https://hacker-news-dotnet-angular.azurewebsites.net](https://hacker-news-dotnet-angular.azurewebsites.net)

