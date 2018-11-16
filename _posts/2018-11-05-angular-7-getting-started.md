---
layout: post
title: Angular 7 Getting Started
date: 2018-11-05
tags: [ Javascript, Angular ]
---

[Angular](https://angular.io/) is a Javascript framework which allows to build reactive single page applications. Angular is written in [TypeScript](https://www.typescriptlang.org/) which brings lot of new features to build more consistent and complex applications. However, this is not supported by browser, so we'll compile from TypeScript to JavaScript.

# Installation

First of all, we are going to install Angular. We'll need node and the package manager *npm* installed in our computer.

**- Install Angular CLI:**
```
> npm install -g @angular/cli
```

Output:
```
/usr/local/bin/ng -> /usr/local/lib/node_modules/@angular/cli/bin/ng
+ @angular/cli@7.0.4
updated 1 package in 7.28s
```

**- *ng* command for new projects:**
```
ng new my-first-app
```

For this project, we chose not to install Angular Routing and use CSS for stylesheets.

Finally, run the app using:

```
> cd my-first-app
> ng serve
```

**- Install Bootstrap for styling:**
```
> npm install --save bootstrap@3
```

And then edit "angular.json" file to add the bootstrap library:

```json
"styles": [
"node_modules/bootstrap/dist/css/bootstrap.min.css",
"src/styles.css"
],
```

# Introduction

Angular is a multi-modular framework. Each module is composed by components. The module used to be in "src/<my-module>". So, for the new module "app", we add "src/app/app.module.ts":

```js
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';

import { AppComponent } from './app.component';

@NgModule({
  declarations: [
    AppComponent
  ],
  imports: [
    BrowserModule
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

Where:
- *declarations*: register our custom components
- [*imports*](https://angular.io/guide/feature-modules): register common modules in every component
- [*providers*](https://angular.io/guide/providers): utilities to instruct Angular how to resolve a dependency 
- *bootstrap*: components that need to be available at startup

This module imports only one component in "src/app/app.component.ts":

```js
import { Component } from '@angular/core';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent {
  myField = 'my-first-app';
}
```

Note the *selector* field is "app-root" needs to match with above element. The *selector* field works like a CSS matching, we can select elements with a class by ".myClass" or by an ID "#myId". 

Moreoever, also note the real HTML is defined in the *templateUrl* field with "app.component.html" which content is:

```html
<h1>{{ myField }}</h1>
```

Finally we need to register our module in "main.ts" typescript file:

```ts
import { enableProdMode } from '@angular/core';
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';

import { AppModule } from './app/app.module';
import { environment } from './environments/environment';

if (environment.production) {
  enableProdMode();
}

platformBrowserDynamic().bootstrapModule(AppModule)
  .catch(err => console.error(err));
```

**The *bootstrapModule* field tells Angular that this module needs to be available at startup.**

So, the HTML will replace the element "<app-root>" in the "index.html":

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>MyFirstApp</title>
  <base href="/">

  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="icon" type="image/x-icon" href="favicon.ico">
</head>
<body>
  <app-root></app-root>
</body>
</html>
```

When we run our Angular application, the server will append the JS sources at the end of the "index.html".

# Basics

## Components

Let's create a new component that is only used in another component, so it's not needed at startup (it's not a bootstrap component).

Let's write a *ServerComponent* component:

*src/app/server/server.component.ts*
```ts
import { Component } from "@angular/core";

@Component({
    selector: 'app-server',
    templateUrl: './server.component.html'
})
export class ServerComponent {
}
```

*src/app/server/server.component.html*
```html
<p class="myStyle">Hello New Component</p>
```

We can achieve the same without adding the new HTML file by the *template* field instead of the *templateUrl* field as:

*src/app/server/server.component.ts*
```ts
import { Component } from "@angular/core";

@Component({
    selector: 'app-server',
	template: `
	<p class="myStyle">Hello New Component</p>
	`
})
export class ServerComponent {
}
```

And we need to edit the module that we'll use our new *ServerComponent* component, the *AppModule* module:

*src/app/app.module.ts*
```ts
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';

import { AppComponent } from './app.component';
import { ServerComponent } from './server/server.component'; // declare our new component

@NgModule({
  declarations: [
    AppComponent, ServerComponent // add our new component
  ],
  imports: [
    BrowserModule
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

And let's use our component in the existing component:

```html
<h1>{{ myField }}</h1>
<hr>
<app-server></app-server>
```

What about styling? Let's customize a new style for our new component:

*src/app/server/server.component.ts*
```ts
import { Component, OnInit } from "@angular/core";

@Component({
    selector: 'app-server',
    templateUrl: './server.component.html',
    styleUrls: [ './server.component.css' ] // add new style
})
export class ServerComponent {
}
```

*src/app/server/server.component.css*
```css
.myStyle {
    color: red;
}
```

And again this is equivalent to:

*src/app/server/server.component.ts*
```ts
import { Component, OnInit } from "@angular/core";

@Component({
    selector: 'app-server',
    templateUrl: './server.component.html',
	styles: [ `
		.myStyle {
			color: red;
		}
	` ] // add new style
})
export class ServerComponent {
}
```

### View Encapsulation
All the styles defined in the *@Component.styles* field will only take into account to the current component. Therefore, there is not inheritance here as we could expect in CSS. We can disable the view encapsulation as:

```ts
import { Component, ViewEncapsulation } from "@angular/core";

@Component({
    selector: 'app-server',
    templateUrl: './server.component.html',
    styleUrls: [ './server.component.css' ],
    encapsulation: ViewEncapsulation.None // Emulated is set by default.
})
export class ServerComponent {
}
```

### Extra: Create Template Component Using CLI

This command will create all the required files for a new component within our "src/app" folder:

```
> ng generate component myComponentName
```

## DataBinding

**- String Interpolation:**
If our variable in the componet is other than String, the data binding will format it into a String automatically, that's why is called "String Interpolation".
*src/app/server/server.component.html*

```html
<p>Hello {{ name }}</p>
<p>Your age is {{ age }}</p>
<p>So you're {{ isOld() }}</p>
```

*src/app/server/server.component.ts*
```ts
// ...
export class ServerComponent {
	name = 'Jose';
	age: number = 32;
	
	isOld() {
		return this.age > 40 ? "old": "still young";
	}
}
```

We can still manipulate the output after string interpolation. Imagine we output milliseconds in string that represents a date, how could we parse the milliseconds in a proper date format? This is what [Pipe](https://angular.io/guide/pipes) is for:

```html
<p>My birthday is {{ birthdayInMillis | date:"dd/MM/yyyy" }} </p>
```

Pipes are like transformers we need to manipulate an output. We can chain as many of these transformers as we need. Also, see how we can build our custom pipes [here](https://angular.io/guide/pipes#custom-pipes).

**- Property Binding:**
This is used to manipulate DOM properties in HTML elements like *class*, *style*, ... As an exmple, we'll disable a button:

*src/app/property-binding/property-binding.component.html*
```html
<button [disabled]="allowButtonDisabled">Add</button>
```

Note that *allowButtonDisabled* is an expression, so we can write things like "!allowButtonDisabled" or "allowButtonDisabled != true ? false : true" or invoke a method of the component. 

*src/app/property-binding/property-binding.component.ts*
```ts
// ...
export class PropertyBindingComponent {
  allowButtonDisabled = true;

  constructor() {
    setTimeout(() => {
      this.allowButtonDisabled = false;
    }, 4000);
  }

}
```

[This list](https://developer.mozilla.org/en-US/docs/Web/API/Element) contains all the properties we can bind.

**- Event Binding:**
This method is used to map the events in a HTML element to the controller.

*src/app/event-binding/event-binding.component.html*
```html
<p> {{ eventBindingWorking }}</p>

<button (click)="onMakeItWork()">Make it work</button>
```

*src/app/event-binding/event-binding.component.ts*
```ts
// ...
export class EventBindingComponent {

  eventBindingWorking = "Not working...";

  onMakeItWork() {
    this.eventBindingWorking = "It worked!";
  }
}
```

[This list](https://developer.mozilla.org/en-US/docs/Web/Events) contains all the events we can bind.

**- Local References to View:**

We also can bind HTML elements to our controller using # + any name:

```html
<input type="text" #serverNameInput/>
<button (click)="myMethod(serverNameInput)">Click</button>
```

```ts
myMethod(serverNameInput) {
  console.log("Local reference is:" + serverNameInput.value);
}
```

Or we can bind a property in our component using *@ViewChild* annotation:

```html
<input type="text" #serverNameInput/>
<button (click)="myMethod()">Click</button>
```

```ts
@ViewChild() serverNameInput; // we could type this local to HtmlInputElement in Angular Core.

myMethod() {
  console.log("Local reference is:" + this.serverNameInput.value);
}
```

If we want to bind a collection of HTML elements, we need to use the annotation *@ViewChildren* insteads:

```ts
@ViewChild() serverNameInputs;
```

And finally, we could bind also the component itself via the annotation *@ContentChild* or *@ContentChildren*:
```ts
@ContentChild() myComponent: MyComponentClass;
```

**- Two-Way-Binding:**

This method is used to combine in and out bindings:

*src/app/two-way-binding/two-way-binding.component.html*

```html
<p>{{ serverName }}</p>
<input type="text" [(ngModel)]="serverName">
```

*src/app/two-way-binding/two-way-binding.component.ts*
```ts
// ...
export class TwoWayBindingComponent {
  serverName = "Testserver";
}
```

The field *serverName* will be updated in both directions with no more logic behind. 

**We need to import the *FormsModule* library in the App module:**

*src/app/app.module.ts*
```ts
// ...
import { FormsModule } from '@angular/forms'; 

@NgModule({
  // ...
  imports: [
	// ...
	FormsModule 
  ],
  // ...
})
export class AppModule { }
```

## Directives

The directives are a way to manipulate the DOM object in HTML. There are three types of directives:

- [Attribute Directives](https://angular.io/guide/attribute-directives)
- [Structural Directives](https://angular.io/guide/structural-directives)
- [Components Directive](https://angular.io/guide/dynamic-component-loader)

Let's see a few examples.

**- ngIf:**

This directive is used to hide or show an element if the condition matches. This directive is a structural directive so for using it, we need to mark it as "*ngIf".

*src/app/directives/directives.component.html*
```html
<input type="text" [(ngModel)]="serverName">
<p *ngIf="serverName != ''">Server Name is {{ serverName }}</p>
```

*src/app/directives/directives.component.ts*
```ts
// ...
export class DirectivesComponent {
  serverName = '';
}
```

What about the else branch? This is how we can use a template in case of not matching the else condition:

*src/app/directives/directives.component.html*
```html
<input type="text" [(ngModel)]="serverName">
<p *ngIf="serverName != ''; else noServer">Server Name is {{ serverName }}</p>
<ng-template #noServer>
  <p>No server!</p>
</ng-template>
```

**- ngStyle:**

This is to define a style depending on some criteria. This is an attribute directive, so for using it we need to type it as "[ngStyle]".

*src/app/directives/directives.component.html*
```html
<p [ngStyle]="{backgroundColor: serverStatus == 'online' ? 'green' : 'red'}">The server is {{ serverStatus }}</p>
```

*src/app/directives/directives.component.ts*
```ts
// ...
export class DirectivesComponent {
  serverStatus = 'offline';

  constructor() {
    this.serverStatus = Math.random() > 0.5 ? 'online' : 'offline';
  }
}
```

What about if we want to append a CSS class instead? Use *ngClass* attribute directive.

*src/app/directives/directives.component.html*
```html
<p [ngClass]="{ onlineClass: serverStatus === 'online'}">The server is {{ serverStatus }}</p>
```

**- ngFor:**

This is used to display lists. It's a structural component, so for using it, we need to type "*ngFor". This directive will repeat the element where is placed.

*src/app/directives/directives.component.html*
```html
<button (click)="addNewServer()">Add Server</button>
<p *ngFor="let server of servers">
  {{server}}
</p>
```

*src/app/directives/directives.component.ts*
```ts
// ...
export class DirectivesComponent {
  servers = [];
  serverIndex = 0;

  addNewServer() {
    this.serverIndex++;
    this.servers.push('Server ' + this.serverIndex);
  }
}
```

## Binding Custom Components

**- Via Property Binding:**

We can use the above directives also in our custom components like for example to display a list of posts.

Let's imagine we want to display these posts using a custom component:

*.../post-list/post/post.component.html*
```html
<div class="row">
  <h2>{{ post.title }}</h2>
</div>
<div class="row">
    <div class="col-xs-12">
        <p> {{ post.body }}</p>
    </div>
</div>
```

*.../post-list/post/post.component.ts*
```ts
import { Component, OnInit, Input } from '@angular/core';
import { Post } from '../post.model';
// ...
export class PostComponent {}
  @Input() post: Post;
}
```

Note the *@Input* decorator. This is how we specify in Angular that this component requires an input.

And then let's display the list of posts by using a *ngFor* directive:

*.../post-list/post-list.component.html*
```html
<div class="row">
  <app-post *ngFor="let post of posts" [post]="post"></app-post>
</div>
```

Note the property binding *[post]*. The *post* event needs to match with the input field in the component handler.

*.../post-list/post-list.component.ts*
```ts
// ..
export class PostListComponent {

  posts: Post[] = [
    new Post('Title', 'Description')
  ];
}
```

**- Via Event Binding:**

Let's add new posts dynamically using a new component "post-edit".

*.../post-edit/post-edit.component.html*
```html
<div class="row">
  <label>Title</label>
  <input type="text" [(ngModel)]="newPost.title">
</div>

<div class="row">
  <label>Post</label>
  <input type="text" [(ngModel)]="newPost.body">
</div>


<button (click)="onPostAdded()">Add POST</button>
```

*.../post-edit/post-edit.component.ts*
```ts
import { Component, EventEmitter, Output } from '@angular/core';
import { Post } from '../post-list/post.model';

//...
export class PostEditComponent {

  @Output() postCreated = new EventEmitter<Post>();
  
  newPost = new Post("", "");

  onPostAdded() {
    this.postCreated.emit(this.newPost);
  }

}
```

Note *@Output* decorator is used to mark the outputs of a component. The *EventEmitter* class from the Angular core is used as an event binder from outside. We call to *emit* to send the data.

*.../post-list/post-list.component.html*
```html
/* .. */
<div class="row">
  <app-post-edit (postCreated)="onPostAdded($event)"></app-post-edit>
</div>
```

And this is how we subscribe to an event called *postCreated* and invoke a method *onPostEvent*:

*.../post-list/post-list.component.ts*
```ts
// ..
export class PostListComponent {

  posts: Post[];

  onPostAdded(newPost : Post) {
    this.posts.push(newPost);
  }
}
```

# Component Lifecycle

| Step  | Method | Description |
| ------------- | ------------- | ------------- |
| 1 | **ngOnChanges** | Called after a bound input property changes. |
| 2 | **ngOnInit** | Called once the component is initialized. Display not ready yet. Properties ready. |
| 3 | **ngDoCheck** | Called during every change detection run. |
| 4 | **ngAfterContentInit** | Called after content (ng-content) has been projected into view. |
| 5 | **ngAfterContentChecked** | Called every time the projected content has been checked. |
| 6 | **ngAfterViewInit** | Called after the component's view (and child views) has been initialized. |
| 7 | **ngAfterViewChecked** | Called every time the view (and child views) have been checked. |
| 8 | **ngOnDestroy** | Called once the component is about to be destroyed. |

In order to get some practice:

```ts
import { Component, OnInit, OnChanges, SimpleChanges, DoCheck, AfterContentInit, AfterContentCheck, AfterViewInit, AfterViewChecked, OnDestroy } from '@angular/core';

export class MyComponent implements OnInit, OnChanges, DoCheck, AfterContentInit, AfterContentChecked, AfterViewInit, AfterViewChecked, OnDestroy {
  constructor() {
    console.log('constructor called');
  }

  ngOnInit() {
    console.log('ngOnInit called');
  }

  ngOnChanges(changes: SimpleChanges) {
    console.log('ngOnChanges called with: ' + changes);
  }

  ngDoCheck() {
    console.log('ngDoCheck called');
  }

  ngAfterContentInit() {
    console.log('ngAfterContentInit called');
  }

  ngAfterContentChecked() {
    console.log('ngAfterContentChecked called');
  }

  ngAfterViewInit() {
    console.log('ngAfterViewInit called');
  }

  ngAfterViewChecked() {
    console.log('ngAfterViewChecked called');
  }

  ngOnDestroy() {
    console.log('ngOnDestroy called');
  }
} 
```

# Custom Directives

In the Basics section, we introduced the concept of directives. Angular has attributes directives to manipulate the attributes in a HTML element and structural directives to manipulate the structure of the hTML itself. 

## Attribute Directive

Let's see how we can create custom attribute directive. Our new directive will highlight a text:

First, we'll use the *ng* command to generate the required files:

```
ng generate directive basic-highlight
```

This command will generate the next file:

*src/app/basic-highlight.directive.ts*
```ts
import { Directive, ElementRef, OnInit, Renderer2 } from '@angular/core';

@Directive({
  selector: '[appBasicHighlight]'
})
export class BasicHighlightDirective implements OnInit {
  
  constructor(private elementRef: ElementRef, private renderer: Renderer2) {
  }

  ngOnInit(): void {
    this.renderer.setStyle(this.elementRef.nativeElement, 'background-color', 'green');
  }
}
```

*Renderer2* element works as a proxy between directives and elements. We should use it instead of manipulating directly the *elementRef*. More about *Rendered2* in [here](https://alligator.io/angular/using-renderer2/).

And the command will also register the directive in the main module:

*src/app/app.module.ts*
```ts
import { NgModule } from '@angular/core';

import { BasicHighlightDirective } from './basic-highlight.directive';

// ...
@NgModule({
  declarations: [
  // ...
    BasicHighlightDirective
  ],
  // ...
})
export class AppModule { }
```

Note we are marking with the *@Directive* annotation and the selector which works as in components: Angular will render this directive in the elements marked as this selector:

*in any html*
```html
<p appBasicHighlight>This will be highlighted!</p>
```

What if we want to make our directive smarter with some logic? We can achive this with the *@HostListener* annotation. Let's see an example to listen the mouse events:

*src/app/basic-highlight.directive.ts*
```ts
import { Directive, ElementRef, OnInit, Renderer2, HostListener } from '@angular/core';

@Directive({
  selector: '[appBasicHighlight]'
})
export class BasicHighlightDirective implements OnInit {
  
  constructor(private elementRef: ElementRef, private renderer: Renderer2) {
  }

  // ...

  @HostListener('mouseover') highlight(eventData: Event) {
    this.renderer.setStyle(this.elementRef.nativeElement, 'background-color', 'green');
  }

  @HostListener('mouseleave') unhighlight(eventData: Event) {
    this.renderer.setStyle(this.elementRef.nativeElement, 'background-color', 'blue');
  }
}
```

Another approach is to bind directly the attribute we want to manipulate:

```ts
import { Directive, HostBinding, HostListener } from '@angular/core';

@Directive({
  selector: '[appBasicHighlight]'
})
export class BasicHighlightDirective {

  @HostBinding('style.backgroundColor') backgroundColor: string = 'blue';

  // ...

  @HostListener('mouseover') highlight(eventData: Event) {
    this.backgroundColor = 'green';
  }

  @HostListener('mouseleave') unhighlight(eventData: Event) {
    this.backgroundColor = 'blue';
  }
}
```

What if we want to make the highlight color configurable? Let's add *@Input* annotation to configure our directive!

```ts
import { Directive, HostBinding, HostListener, Input } from '@angular/core';

@Directive({
  selector: '[appBasicHighlight]'
})
export class BasicHighlightDirective {

  @Input() highlightColor: String = 'green';
  @HostBinding('style.backgroundColor') backgroundColor: string = this.highlightColor;

  // ...

  @HostListener('mouseover') highlight(eventData: Event) {
    this.backgroundColor = this.highlightColor;
  }

  @HostListener('mouseleave') unhighlight(eventData: Event) {
    this.backgroundColor = 'blue';
  }
}
```

In HTML:
```html
<p appBasicHighlight [highlightColor="'yellow'"]>This will be highlighted!</p>
```

Or we can define a default property as:

```ts
import { Directive, HostBinding, HostListener, Input } from '@angular/core';

@Directive({
  selector: '[appBasicHighlight]'
})
export class BasicHighlightDirective {

  @Input('appBasicHighlight') highlightColor: String = 'green';
  @HostBinding('style.backgroundColor') backgroundColor: string = this.highlightColor;

  // ...
}
```

In HTML:
```html
<p [appBasicHighlight]="'yellow'"]>This will be highlighted!</p>
```

## Structural Directive

Again, we use the *ng* command to generate the required files:

```
ng generate directive unless
```

Then our structural directive:

*src/app/basic-highlight.directive.ts*
```ts
import { Directive, ViewContainerRef, Input, TemplateRef } from '@angular/core';

@Directive({
  selector: '[appUnless]'
})
export class UnlessDirective {

  @Input('appUnless') set unless(condition: boolean) {
    if (!condition) {
      this.vcRef.createEmbeddedView(this.templateRef);
    } else {
      this.vcRef.clear();
    }
  }

  constructor(private templateRef: TemplateRef<any>, private vcRef: ViewContainerRef) { }

}
```

And how we use it in HTML:

*in any html*
```html
<div *appUnless="myCondition">
  <p>It worked!</p>
</div>
```

# Services

The services are common functionality we want across our components and are used typically for gathering data or for utilities. 

Let's create a logging service. First, we'll use the *ng* command:

```
ng generate service logging
```

This command will generate our service:

*src/main/app/logging.service.ts*
```ts
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root' // tells the scope of the service: root (Singleton) or any concrete component.
})
export class LoggingService {

  debug(msg: string) {
    console.log('My Service: ' + msg);
  }
}
```

Note the *@Injectable* annotation which will make our new service available in the components scope:

*any component.ts*
```ts
import { Component } from '@angular/core';

import { LoggingService } from '../logging.service';

@Component(
  // ..
)
export class MyComponent {
  constructor(private loggingService: LoggingService) {
  }

  aMethod() {
    loggingService.debug('My Message!');
  }
}
```

# Routing

Most of the sites are composed by several pages or child pages. Routing is a way to navigate from one page to another within the same site. Let's see how to do this in Angular:

*src/app/app.module.ts*
```ts
// ...
import { Routes, RouterModule } from '@angular/router';

// Routes mapping
const appRoutes: Routes = [
  { path: '', component: PostListComponent },// static
  { path: 'posts', component: PostListComponent, children: [
    { path: ':id', component: PostComponent } // nested & dynamic
  ] }, 
  { path: 'about', component: AboutsComponent } // static
];

@NgModule({
  declarations: [
    AppComponent,
    PostListComponent,
    AboutComponent
  ],
  imports: [
    // ...
    RouterModule.forRoot(appRoutes) // We use RouterModule to register our routing
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

Then we need to replace the view of our main component *AppComponent* with a special placeholder *router-outlet* and **also to the components with nested paths**:

*src/app/app.component.html*
```html
<app-header></app-header>
<div class="container">
    <div class="row">
        <div class="col-md-12">
            <router-outlet></router-outlet> <!-- Router placeholder -->
        </div>
    </div>
</div>
```

Angular will replace the *router-outlet* placeholder with the view of the component we defined above.

What about if we have a navigation bar? As simple as:

*src/app/header/header.component.html*
```html
<ul class="nav navbar-nav">
  <li><a href="/posts">Noticias</a></li>
  <li><a href="/about">Sobre Nosotros</a></li>
</ul>
```

Note that this will reload the browser to render the new page. In case we don't want to reload the browser, Angular provides *routerLink* property binding:

*src/app/header/header.component.html*
```html
<ul class="nav navbar-nav">
  <!-- ... -->
  <li>
    <a [routerLink]="['/posts',1]" 
      [queryParams]="{key:'value'}">
      Go to Post with ID = 1
    </a>
  </li>
</ul>
```

Bear in mind we are using absolute paths: "/" + the path. Without "/", we would be navigating relatively from the current path. 

Where *routerLink* is used to navigate absolutely or relatively and *queryParams* to append optional parameters in the url.

How can we navigate programmatically? Let's see an example:

*any component.ts*
```ts
// ...
import { Routes, ActivatedRoute } from '@angular/router';

@Component({
  // ...
})
export class MyComponent {

  constructor(private router: Router, // this is the service
    private route: ActivatedRoute) // this is to know in what route we're
  {  }

  navigateToAbout() {
    this.router.navigate(['/about']); // absolute path
  }

  navigateToPost(postId) {
    this.router.navigate(['post', postId], 
      { relativeTo: this.route,
      queryParams: { key : 'value' }
    }); // relative path
  }
}
```

Let's see how we can get path params and query values:

*src/app/post-list/post/post.component.ts*
```ts
// ...
import { Routes, ActivatedRoute } from '@angular/router';

@Component({
  // ...
})
export class PostComponent implements OnInit {
  postId : String;

  constructor(private route: ActivatedRoute) {  }

  ngOnInit() {
    this.postId = this.route.snapshot.params['id']; // this is how we get path params
    this.other = this.route.snapshot.queryParams['key'];

    this.route.params.subscribe((params: Param) => { this.postId = params['id'] }); // Reactive approach 
    this.route.queryParams.subscribe((params: Param) => { this.postId = params['id'] }); // Reactive approach 
  }
}
```

**- Manipulate the DOM depending on the current path:**

There are a bunch of options to enable/disable or manipulate the DOM depending on the current path. Refer to [the Angular documentation](https://codecraft.tv/courses/angular/routing/navigation/#_routerlinkactive) for more details.

**- How to preserve our query params among different paths:**

*any component.ts*
```ts
// ...

@Component({
  // ...
})
export class MyComponent {

  // ..

  navigateToEdit() {
    this.router.navigate(['edit'], 
      { queryParamsHandling: "merge" } // keep query params!
    }); 
  }
}
```

**- Redirect to:**

How can we redirect to a path according to any regular expression? In our route mapping:

*src/app/app.module.ts*
```ts
// ...
import { Routes, RouterModule } from '@angular/router';

// Routes mapping
const appRoutes: Routes = [
  { path: '', component: PostListComponent },
  { path: 'about', component: AboutsComponent },
  { path: '**', redirectTo: '/not-found', pathMatch: 'full', data: { message: 'Page Not Found' } } 
];

@NgModule({
  // ...
})
export class AppModule { }
```

**Order is important!** If we move the "not found" rule to the first place, it will navigate to the "not found" page always because the rule always applies.

Note we are passing some static data "message" to our "not-found" view througout the *data* field. For dynamic data, we have to use the *resolve* field and a service.

**- How can we protect our paths:**

This is done with the *CanActivate* interface by Angular. We need to create a service and use the *canActivate* field when configuring our routes. More info [here](https://angular.io/api/router/CanActivate). The similar with the nested paths with *CanActivateChild* interface.

# Custom Module

Let's move out the routing mapping rules into a separate module. Create a AppRoutingModule as:

*src/app/app-routing.module.ts*
```ts
import { NgModule } from '@angular/core';

import { Routes, RouterModule } from '@angular/router';
// ...

const appRoutes: Routes = [
  { path: '', component: PostListComponent },
  { path: 'posts', component: PostListComponent },
  { path: 'about', component: AboutsComponent }
];

@NgModule({
  imports: [
    RouterModule.forRoot(appRoutes)
  ],
  exports: [ RouterModule ] // Anything we want to export
})
export class AppRoutingModule { }
```

And how to import our custom module as:

*src/app/app.module.ts*
```ts
// ...
import { AppRoutingModule } from './app-routing.module';

@NgModule({
  declarations: [
    // ...
  ],
  imports: [
    // ...
    AppRoutingModule // Import our custom module
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

# Forms

Forms is the common way to send entered user data to servers. We can play with forms in Angular in two ways: template driven or reactive approach.

## Template Driven

In order to use all the features Angular provides, make sure we import the *FormsModule* module:

*src/app/app.module.ts*
```ts
// ...
import { FormsModule } from '@angular/forms'; 

@NgModule({
  // ...
  imports: [
  	// ...
	  FormsModule 
  ],
  // ...
})
export class AppModule { }
```

The template or the view will drive the data we send in the form via the attribute binding *ngModel* and the event binding *ngSubmit*:

*any.component.html*
```html
<form (ngSubmit)="onSubmit(f)" #f="ngForm">
  <div>
    <label for="username">Username</label>
    <input type="text" id="username" ngModel name="user"/>
  </div>

  <div>
    <label for="email">Username</label>
    <input type="email" id="email" ngModel name="email"/>
  </div>

  <button type="submit">Submit</button>
</form>
```

*any.component.ts*
```ts
// ..
@Component(
  // ... 
)
export class AnyComponent {
  onSubmit(form: NgForm) {
    console.log(form);

    form.reset(); // this will clear the form data.
  }
}
```

We usually want to validate our data before submit it to the server. Refer to the [Angular Documentation](https://angular.io/guide/form-validation) about validations. [This](https://angular.io/api/forms/Validators) is the list of all validators we have in Angular.

## Reactive Approach

The reactive approach will create a form programmatically. Instead of using the *FormsModule*, we need the *ReactiveFormsModule*:

*src/app/app.module.ts*
```ts
// ...
import { ReactiveFormsModule } from '@angular/forms'; 

@NgModule({
  // ...
  imports: [
  	// ...
	  ReactiveFormsModule 
  ],
  // ...
})
export class AppModule { }
```

*any.component.ts*
```ts
import { FormGroup, FormControl } from '@angular/forms';

// ..
@Component(
  // ... 
)
export class AnyComponent implements OnInit {

  form: FormGroup;

  ngOnInit() {
    this.form = new FormGroup({
      'user': new FormControl(null),
      'email': new FormControl(null)
    });
  }

  onSubmit() {
    console.log(form);

    form.reset(); // this will clear the form data.
  }
}
```

*any.component.html*
```html
<form (ngSubmit)="onSubmit()" [formGroup]="form">
  <div>
    <label for="username">Username</label>
    <input type="text" id="username" formControlName="user"/>
  </div>

  <div>
    <label for="email">Username</label>
    <input type="email" id="email" formControlName="email"/>
  </div>

  <button type="submit">Submit</button>
</form>
```

# Observables

Angular uses the concept of Observable along to its framework. To get familiar with the Observable concept, refer to [the documentation here](https://angular.io/guide/observables). Basically, Angular uses the [RxJS framework](https://rxjs-dev.firebaseapp.com/). If we go to the package.json file:

```json
{
  "name": "my-first-app",
  "version": "0.0.0",
  "scripts": {
    // ...
  },
  "private": true,
  "dependencies": {
    // ...
    "rxjs": "~6.3.3" // RxJS library
  },
  "devDependencies": {
    // ...
  }
}
```

Let's write a custom observable that emits a couple of numbers and then fails:

*any component.ts*
```ts

import { Observable } from 'rxjs/Observable';
import 'rxjs/Rx';
import { Observer } from 'rxjs/Observer';
import { Subscription } from 'rxjs/Subscription';

// ...
export MyComponent implements OnInit, OnDestroy {

  susbcription: Subscription;

  constructor() {}

  ngOnInit() {
    const myNumbers = Observable.create((observer: Observer<number>) => {
      
      setTimeout(() => {
        observer.next(1);
      }, 1000);

      setTimeout(() => {
        observer.next(2);
      }, 1000);

      setTimeout(() => {
        observer.error('something failed!');
      }, 5000);
    })); 

    this.susbcription = myNumbers.subscribe(
      // handler
      (number: Number) => {
        console.log(number);
      },
      // errors: invoked when observer.error('something failed!'):
      (error: srting) => {
        console.log(error);
      },
      // completed: invoked when observer.complete();
      () => {
        console.log('completed!');
      }
    );
  }

  ngOnDestroy() {
    this.susbcription.unsubscribe();
  }
}
```

**It's important to unsubscribe when the component exits.** Observables work as streams in Java, we can transform data or merge different data. Go go [the RxJS Observable documentation](http://reactivex.io/rxjs/class/es6/Observable.js~Observable.html) to go deep in what we can do with observables.

# HttpClient

A new [HttpClient](https://angular.io/guide/http) was introduced in Angular 6 which deprecates [the old Http Request](https://angular.io/api/common/http/HttpRequest).

We need to enable it in our module:

*src/app/app.module.ts*
```ts
// ...
import { BrowserModule }    from '@angular/platform-browser';
import { HttpClientModule } from '@angular/common/http';

@NgModule({
  // ...
  imports: [
  	// ...
    BrowserModule,
    HttpClientModule // after BrowserModule!
  ],
  // ...
})
export class AppModule { }
```

Create a service with the common HTTP config:

src/app/config.data.service.ts*
```ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Injectable()
export class ConfigDataService {
  constructor(private http: HttpClient) { }
}
```

Then, the field *http* exposes the HTTP methods get, post, put, ... The important keynote here is that the response is [a promise](https://developer.mozilla.org/es/docs/Web/JavaScript/Referencia/Objetos_globales/Promise). 

# Conclusion

What a large post! I really enjoyed learning the basics of Angular and there are still many things I didn't cover. However, I would like to focus and congratulate the really good documentation and community support behind Angular. I can't imagine something it's not replied already in StackOverflow. About React or Angular, it depends, but for me, I like to completely separate views to the logic even if I need to lose a bit of reactive. Yet this is my personal guess!

See [my Github repository](https://github.com/Sgitario/angular-getting-started) for a full example.