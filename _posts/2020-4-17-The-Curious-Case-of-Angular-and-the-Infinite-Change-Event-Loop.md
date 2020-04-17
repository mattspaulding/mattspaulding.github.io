---
layout: post
title: The Curious Case of Angular and the Infinite Change Event Loop
image: /images/dotnet-core-angular.jpg
excerpt: Angular change detection is a mystical wonder. This is the behind-the-curtain magic that makes Angular "just work". Today we pull back the curtain on Angular change events and discover when they don't "just work" (and how to fix them).
---

![dotnet-core-angular]({{ site.baseurl }}/images/dotnet-core-angular.jpg)

Angular change detection is a mystical wonder. This is the behind-the-curtain magic that makes Angular "just work". Today we pull back the curtain on Angular change events and discover when they don't "just work" (and how to fix them).

I encourage you to follow along...

> See the final project in the [repository](https://github.com/mattspaulding/angular-infinite-event-loop)

## Built with

- Angular v9.1

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
   ng new infinite-loop --interactive=false
   ```

1. Navigate into the repository

   ```sh
   cd infinite-loop
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

   ![initial-start]({{site.baseurl}}/images/2020-4-17-The-Curious-Case-of-Angular-and-the-Infinite-Change-Event-Loop/initial-start.png)

## The Code

All started up? Great! Now let's put some of our code in.

1. Kill the server

   ```sh
   CTRL + C
   ```

1. Generate a new component named "hello"

   ```sh
   ng generate component hello
   ```

1. In `src/app/hello.component.html`

   ```html
   <p *ngIf="!isLoggedIn()">Are you there?</p>
   <h1 *ngIf="isLoggedIn()">Hello {{user?.first}}!</h1>
   ```

1. In `src/app/hello.component.ts`

   ```ts
   import { Component, Input, AfterViewChecked } from "@angular/core";

   @Component({
     selector: "hello",
     templateUrl: "./hello.component.html",
     styleUrls: ["./hello.component.css"],
   })
   export class HelloComponent implements AfterViewChecked {
     @Input() user: any;
     constructor() {}

     ngAfterViewChecked() {
       console.count("ngAfterViewChecked");
     }

     isLoggedIn() {
       if (this.user?.first) return true;
       else return false;
     }
   }
   ```

1. In `src/app/app.component.html`

   ```html
   <hello [user]="user"></hello> <button (click)="login($event)">Login</button>
   ```

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
       this.user.first = "Bob";
       this.user.last = "Smith";
     }
   }
   ```

1. Start up the project again

   ```sh
   npm start
   ```

![default-fine]({{site.baseurl}}/images/2020-4-17-The-Curious-Case-of-Angular-and-the-Infinite-Change-Event-Loop/default-fine.gif)

This is how it should look. Clicking the login button calls the login method which changes the user's name. The `user` object is passed into the `hello` component. The component checks for a first name and displays if one exists. In the hello component we log every time the `ngAfterViewChecked` Angular lifecycle hook is invoked.

### Infinite Event Loop

Now, let's say we want to do something 1 second after the hello component checks for login.

1. In `src/hello/hello.component.ts`

```ts
import { Component, Input, AfterViewChecked } from "@angular/core";

@Component({
  selector: "hello",
  templateUrl: "./hello.component.html",
  styleUrls: ["./hello.component.css"],
})
export class HelloComponent implements AfterViewChecked {
  @Input() user: any;
  constructor() {}

  ngAfterViewChecked() {
    console.count("ngAfterViewChecked");
  }

  isLoggedIn() {
    setTimeout(() => {
      // do something after 1 second
    }, 1000);

    if (this.user?.first) return true;
    else return false;
  }
}
```

![default-oops]({{site.baseurl}}/images/2020-4-17-The-Curious-Case-of-Angular-and-the-Infinite-Change-Event-Loop/default-oops.gif)

Woopsie! We have just entered an endless event loop and my Macbook Pro gave me third-degree burns on my thighs.

![computer-fire](https://media.giphy.com/g79am6uuZJKSc.gif)

### The Fix

_So what happened?_

Calling `setTimeout` causes a change event to fire. The change event cause the `isLoggedIn()` method to run. Which causes another `setTimeout` and so on and so on. This is the default change detection strategy. To make Angular "just work" it checks for changes on several things. There are a couple things we can do to fix this. And make our application more efficient too.

1. NgZone
2. OnPush

#### NgZone

1. In `src/hello/hello.component.ts`

```ts
import { Component, Input, AfterViewChecked, NgZone } from "@angular/core";

@Component({
  selector: "hello",
  templateUrl: "./hello.component.html",
  styleUrls: ["./hello.component.css"],
})
export class HelloComponent implements AfterViewChecked {
  @Input() user: any;
  constructor(private zone: NgZone) {}

  ngAfterViewChecked() {
    console.count("ngAfterViewChecked");
  }

  isLoggedIn() {
    this.zone.runOutsideAngular(() => {
      setTimeout(() => {
        // do something after 1 second
      }, 1000);
    });

    if (this.user?.first) return true;
    else return false;
  }
}
```

![default-ngzone]({{site.baseurl}}/images/2020-4-17-The-Curious-Case-of-Angular-and-the-Infinite-Change-Event-Loop/default-ngzone.gif)

We wrapped our `setTimeout` with `NgZone.runOutsideAngular`. This tells Angular to not fire a change event for this section of code.

#### OnPush

1. In `src/hello/hello.component.ts`

```ts
import {
  Component,
  Input,
  AfterViewChecked,
  ChangeDetectionStrategy,
} from "@angular/core";

@Component({
  selector: "hello",
  templateUrl: "./hello.component.html",
  styleUrls: ["./hello.component.css"],
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class HelloComponent implements AfterViewChecked {
  @Input() user: any;
  constructor() {}

  ngAfterViewChecked() {
    console.count("ngAfterViewChecked");
  }

  isLoggedIn() {
    setTimeout(() => {
      // do something after 1 second
    }, 1000);

    if (this.user?.first) return true;
    else return false;
  }
}
```

![onpush-oops]({{site.baseurl}}/images/2020-4-17-The-Curious-Case-of-Angular-and-the-Infinite-Change-Event-Loop/onpush-oops.gif)

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

![onpush-fixed]({{site.baseurl}}/images/2020-4-17-The-Curious-Case-of-Angular-and-the-Infinite-Change-Event-Loop/onpush-fixed.gif)

There. Now its working.

But look carefully and you will notice that 1 second after the the login button is pressed there are some more events. This is due to the `setTimeout` returning a second later.

## Conclusion

In conclusion, which is best for solving the infinite event loop, 1) NgZone or 2) OnPush? Plot twist! I choose 3) All of the above.

1. In `src/hello/hello.component.ts`

```ts
import {
  Component,
  Input,
  AfterViewChecked,
  NgZone,
  ChangeDetectionStrategy,
} from "@angular/core";

@Component({
  selector: "hello",
  templateUrl: "./hello.component.html",
  styleUrls: ["./hello.component.css"],
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class HelloComponent implements AfterViewChecked {
  @Input() user: any;
  constructor(private zone: NgZone) {}

  ngAfterViewChecked() {
    console.count("ngAfterViewChecked");
  }

  isLoggedIn() {
    this.zone.runOutsideAngular(() => {
      setTimeout(() => {
        // do something after 1 second
      }, 1000);
    });

    if (this.user?.first) return true;
    else return false;
  }
}
```

![onpush-ngzone]({{site.baseurl}}/images/2020-4-17-The-Curious-Case-of-Angular-and-the-Infinite-Change-Event-Loop/onpush-ngzone.gif)

And now you are a little bit cooler programmer.

![computer-fire](https://media.giphy.com/m2Q7FEc0bEr4I.gif)
