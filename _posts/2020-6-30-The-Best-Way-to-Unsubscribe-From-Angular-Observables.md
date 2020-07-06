---
layout: post
author: Matt Spaulding
section: Technology
published_time: 2020-06-30
modified_time: 2020-06-30
title: The Best Way to Unsubscribe from Angular Observables
image: /images/2020-6-30-The-Best-Way-to-Unsubscribe-From-Angular-Observables/angular-observable-unsubscribe.png
excerpt: The inclusion of RxJS Observable in Angular core is one of the most important additions in Angular's history. It is also one of the most misused. When used improperly, the Observable can open up memory leaks making your application sluggish or causing it to crash. Today we will talk about how to properly clean up observables and the very best way to do it.
---

![angular-infinite-event-loop]({{ site.baseurl }}/images/2020-6-30-The-Best-Way-to-Unsubscribe-From-Angular-Observables/angular-observable-unsubscribe.png)

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
{% raw %}
   ```html
   <p>{{ time }}</p>
   ```
{% endraw %}
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

Clicking the `count` button starts the timer and shows the count. Click `stop` and it stops. Click `count` again and we get a fresh timer starting at zero.

But wait, what is happening in the console?...

### The Memory Leak

Woopsie! When we press `stop`, the observable subscription is not actually stopped. We can see in the cosole that it keeps on keeping on. What's worse is that every new timer that is started continues to run forever. This, my friends is what we call a memory leak.

![its-fine]({{site.baseurl}}/images/2020-6-30-The-Best-Way-to-Unsubscribe-From-Angular-Observables/its-fine.gif)

You might be able to get away with this if your app is simple seconds counter. But if your app is observing services with heavy payloads, this can easily spiral out of control. Here are three ways to unsubscribe from your Angular observables. Good. Better. and Best.

### Good

Manually unsubscribe from each observable.

1. In `src/app/count/count.component.ts`

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

To unsubscribe from an observable subscription, we must create a `Subscription` variable (timer$), assign the subscription to this variable, and then in the `ngOnDestroy` lifecycle hook unsubscribe the subscription. This is fine... and it works. This is Observables 101.

![count-good]({{site.baseurl}}/images/2020-6-30-The-Best-Way-to-Unsubscribe-From-Angular-Observables/count-good.gif)

And we can see this is now behaving as expected. But what if you had 100 observables in this component? We would have to make 100 variables and unsubscribe 100 times. If there were only a better way.

### Better

Implicitly unsubscribe from observables.

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

In this case we create an `unsubscribe$` Subject. In the observable the RxJS `takeUntil` operator is passed the subject and will remain open until the subject is completed. In `ngOnDestroy`, we call the next item if one exists and then completes.

This is a bit more complicated to understand than the first example, but has a nice benefit. We no longer need to create a variable for each observable and no longer need to unsubscribe in the destroy.

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
{% raw %}
```html
<p>{{(timer$ | async)}}</p>
```
{% endraw %}
## Conclusion

In conclusion, all of these are perfectly acceptable ways to unsubscribe from observables. You choose your own destiny.

![choose-your-destiny]({{site.baseurl}}/images/2020-6-30-The-Best-Way-to-Unsubscribe-From-Angular-Observables/choose-your-destiny.gif)