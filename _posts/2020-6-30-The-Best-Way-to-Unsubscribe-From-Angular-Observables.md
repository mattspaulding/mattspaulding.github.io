---
layout: post
author: Matt Spaulding
section: Technology
published_time: 2020-06-30
modified_time: 2020-06-30
title: The Best Way to Unsubscribe from Angular Observables
image: /images/2020-6-30-The-Best-Way-to-Unsubscribe-From-Angular-Observables.png
excerpt: The inclusion of RxJS Observable in Angular core is one of the most important additions in Angular's history. It is also one of the most misused. When used improperly, the Observable can open up memory leaks making your application sluggish or causing it to crash. Today we will talk about how to properly clean up observables and the very best way to do it.
---

![angular-infinite-event-loop]({{ site.baseurl }}/images/2020-6-30-The-Best-Way-to-Unsubscribe-From-Angular-Observables/angular-infinite-event-loop.png)

The inclusion of RxJS Observable in Angular core is one of the most important additions in Angular's history. It is also one of the most misused. When used improperly, the Observable can open up memory leaks making your application sluggish or causing it to crash. Today we will talk about how to properly clean up observables and the very best way to do it.

I encourage you to follow along...

## Built with

- Angular v10.0.0

## Getting Started

### Prerequisites

Get the newest vscode => [https://code.visualstudio.com/download](https://code.visualstudio.com/download)

Get the newest LTS node => [https://nodejs.org](https://nodejs.org)

### Installation

1. Open a terminal where you want the project to be installed

1. Install the Angular CLI globally

   ```sh
   npm install -g @angular/cli
   ```

1. Create a new Angular project

   ```sh
   ng new angular-observable-unsubscribe --interactive=false
   ```

1. Navigate into the repository

   ```sh
   cd angular-observable-unsubscribe
   ```

1. Open the project in vscode

   ```sh
   code .
   ```

1. Pop open the vscode terminal

   > CTRL + `

1. Start up the project

   ```sh
   npm start
   ```

1. Open a Chrome browser here.

   > http://localhost:4200

   ![initial-start]({{site.baseurl}}/images/2020-6-30-The-Best-Way-to-Unsubscribe-From-Angular-Observables/initial-start.png)

## The Code

All started up? Great! Now let's put some of our code in.

1. Kill the server

   ```sh
   CTRL + C
   ```

1. Generate a new component named "count"

   ```sh
   ng generate component count
   ```

1. In `src/app/count/count.component.html`

   ```html
   <p>{{time}}</p>
   ```

1. In `src/app/count/count.component.ts`

  ```ts
  import { Component, OnInit } from '@angular/core';
  import { interval } from 'rxjs';

  @Component({
    selector: 'count',
    templateUrl: './count.component.html',
    styleUrls: ['./count.component.css']
  })
  export class CountComponent implements OnInit {
    time=0;
    constructor() { }

    ngOnInit(): void {
      interval(1000).subscribe(val=>{
        this.time=val;
        console.log(val);
      })
    }
  }
  ```

1. In `src/app/app.component.html`

   ```html
   <button *ngIf="!showCount" (click)="showCount=true">count</button>
   <button *ngIf="showCount" (click)="showCount=false">stop</button>
   <count *ngIf="showCount"></count>
   ```

1. In `src/app/app.component.ts`

   ```ts
   import { Component } from '@angular/core';

    @Component({
      selector: 'app-root',
      templateUrl: './app.component.html',
      styleUrls: ['./app.component.css']
    })
    export class AppComponent {
      showCount=false;
    }
   ```

1. Start up the project again

   ```sh
   npm start
   ```

![count-no-unsubscribe]({{site.baseurl}}/images/2020-6-30-The-Best-Way-to-Unsubscribe-From-Angular-Observables/count-no-unsubscribe.gif)

This is how it should look. Clicking the login button calls the login method which changes the user's name. The `user` object is passed into the `hello` component. The component checks for a first name and displays if one exists. In the hello component we log every time the `ngAfterViewChecked` Angular lifecycle hook is invoked.

### The Memory Leak

Woopsie! We have just entered an endless event loop and my Macbook Pro gave me third-degree burns on my thighs.

![its-fine]({{site.baseurl}}/images/2020-6-30-The-Best-Way-to-Unsubscribe-From-Angular-Observables/its-fine.gif)

### Good

Now, let's say we want to do something 1 second after the hello component checks for login.

1. In `src/app/hello/hello.component.ts`

```ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { interval, Observable, Subscription } from 'rxjs';

@Component({
  selector: 'count',
  templateUrl: './count.component.html',
  styleUrls: ['./count.component.css'],
})
export class CountComponent implements OnInit, OnDestroy {
  time = 0;
  timer$: Subscription;
  constructor() {}

  ngOnInit(): void {
    this.timer$ = interval(1000).subscribe((val) => {
      this.time = val;
      console.log(val);
    });
  }

  ngOnDestroy(): void {
    this.timer$.unsubscribe();
  }
}
```

![count-good]({{site.baseurl}}/images/2020-6-30-The-Best-Way-to-Unsubscribe-From-Angular-Observables/count-good.gif)

### Better

_So what happened?_

Calling `setTimeout` causes a change event to fire. The change event cause the `isLoggedIn()` method to run. Which causes another `setTimeout` and so on and so on. This is the default change detection strategy. To make Angular "just work" it checks for changes on several things. There are a couple things we can do to fix this. And make our application more efficient too.

1. In `src/app/count/count.component.ts`

```ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { interval, Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';

@Component({
  selector: 'count',
  templateUrl: './count.component.html',
  styleUrls: ['./count.component.css'],
})
export class CountComponent implements OnInit, OnDestroy {
  time = 0;
  private unsubscribe$ = new Subject();
  constructor() {}

  ngOnInit(): void {
    interval(1000)
      .pipe(takeUntil(this.unsubscribe$))
      .subscribe((val) => {
        this.time = val;
        console.log(val);
      });
  }

  ngOnDestroy(): void {
    this.unsubscribe$.next();
    this.unsubscribe$.complete();
  }
}
```

### Best

1. In `src/app/count/count.component.ts`

```ts
import { Component } from '@angular/core';
import { interval } from 'rxjs';

@Component({
  selector: 'count',
  templateUrl: './count.component.html',
  styleUrls: ['./count.component.css'],
})
export class CountComponent {
  timer$ = interval(1000);
}
```

1. In `src/app/count/count.component.html`

```html
<p>{{(timer$ | async)}}</p>
```

![onpush-oops]({{site.baseurl}}/images/2020-6-30-The-Best-Way-to-Unsubscribe-From-Angular-Observables/onpush-oops.gif)

We set the Angular change detection strategy from `ChangeDetectionStrategy.Default` (look for changes everywhere) to `ChangeDetectionStrategy.OnPush` which will only detect changes when the `@Input()` has a new object pushed on it. In this case, when there is a new `user` object.

But, whoops. Nothing happened when I clicked the login button. The problem here is that we are directly mutating the existing `user` object.

```ts
  app.component.ts
  ...

  login(e: MouseEvent) {
    this.user.first = 'Bob';
    this.user.last = 'Smith';
  }

  ...
```

With the `OnPush` strategy we must push a new object through the `hello` component. Try this instead.

1. In `src/app/app.component.ts`

   ```ts
   import { Component } from "@angular/core";

   @Component({
     selector: "app-root",
     templateUrl: "./app.component.html",
     styleUrls: ["./app.component.css"],
   })
   export class AppComponent {
     user = { first: "", last: "" };

     login(e: MouseEvent) {
       this.user = { first: "Bob", last: "Smith" };
     }
   }
   ```

![onpush-fixed]({{site.baseurl}}/images/2020-6-30-The-Best-Way-to-Unsubscribe-From-Angular-Observables/onpush-fixed.gif)

There. Now its working.

But look carefully and you will notice that 1 second after the the login button is pressed there are some more events. This is due to the `setTimeout` returning a second later.

## Conclusion

In conclusion, all of these are perfectly acceptable ways to unsubscribe from observables. You choose your own destiny.

![choose-your-destiny]({{site.baseurl}}/images/2020-6-30-The-Best-Way-to-Unsubscribe-From-Angular-Observables/choose-your-destiny.gif)