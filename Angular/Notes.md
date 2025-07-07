
# Table of Contents

- [Introduction](#directives)

- [Section 1](#section-1)

- [Section 2](#section-2)

  

# Injection hierarchy

  

Essentially this follows almost a "waterfall" pattern.

  

Most typical use case in `{providedIn: 'root'}` and this is a singleton. There are a couple of ways you can override this.

  

1 - Add the service to the providers array of a given component. This gives that component it's own instance of the service

  

2 - At module level. Not very common anymore but if you have a module file and set the provider there, then it will be the same instance across the whole module for all components.

  

There are some quirks to this. For example:

  

```typescript

@Component(){

    Providers:[SomeService]

}

export class ComponentOne(){

  

}

  

@Component(){

}

export class ComponentTwo(){

    public someService: SomeService = inject(SomeService)

  

}

```

  

And in the HTML componentOne is a parent of componentTwo

  

```html

<!-- this is the componentOne markup -->

<app-component-two><app-component-two/>

```

  

In this case, `ComponentTwo` will _share_ ComponentOne instance of the service, even though it injects it as if it were targeting the root injector. Hence the "waterfall" aspect of this.

  

# ForRoot vs ForChild

  

Angular should only have one and only one router service. `ForRoot` and `ForChild` are the old way of accomplishing that. By doing in your app module `Router.forRoot(routes)` you are essentially saying these are the top level routes of your application. _You should only do this one time!_. For all children modules you use `Router.forChild()`. This is what tells Angular these are child routes and as such they should use the same instance of Router. In V19 this is simplified and you just need to do it like this if you use modules.

  

# Directives

  

> A tag that adds behavior to whatever it attaches itself to

  

Quite a few things to know here so I'll paste a Directive implementation below and comment it

  

```typescript

@Directive({

  selector: '[appRandom]',

  standalone: true, //can be standalone or part of a module

  // all the stuff you do here in the "host" you can also do with @HostBinding as @HostListener

  host: {

    '[attr.disabled]': 'disabled || null', //this is one way of attaching things to the host element

    '(click)': 'onClick()' // good for attributes as well as events

  },

})

export class RandomDirective {

  

  // this is a way of getting a reference to the element you are attaching the directive too

  private el = inject(ElementRef)

  

  //you can use renderer to do things on the host like setting a style of whatever. Although this to me defeats the purpose of having a directive

  private renderer = inject(Renderer2)

  

  // "This is the value the host must use for XYZ"

  @HostBinding('style.color') public textColor = ''

  

  public color = input.required<string>()

  public disabled = false

  

  // One way of responding to the listener added to the hosts above

  onClick() {

    this.textColor = this.color()

    console.log('diretive click', this.el)

  }

    //...or a second way if you prefer doing events like this

    // note that way you control the events that are passed to a directive is via the array

    // essentially you can be as granular or as specific as you want. You can do event.target or you can do event.target.value.xyz.ttt.aaa

  @HostListener('mouseenter', ['$event.target'])

  public onMouseEnter(event: unknown) {

    alert('🐁')

  }

}

```

  

Angular as also introduced the concept of host directives. These are essentially directives that you attach to a host component element. Syntax is as follows:

  

```typescript

@Component({

  selector: 'app-random-one',

  providers: [],

  imports: [RandomDirective],

  templateUrl: './random-one.component.html',

  styleUrl: './random-one.component.css',

  standalone: true,

  hostDirectives: [

    { directive: RandomDirective, inputs: ['color'] }

  ]

})

export class RandomOneComponent {

  public randomService = inject(RandomServiceService)

  (...)

}

```

  

Note that the value for the inputs is passed from _outside_ the component just like a normal input:

  

```typescript

<app-random-one [color]='blue'></app-random-one>

```

  
  

# Lifecycle hooks

  

8 Lifecycle Hooks:

  

| Lifecycle Hook            | Purpose                                      | Invocation Frequency |

|---------------------------|----------------------------------------------|----------------------|

| `ngOnChanges()`          | Detect input property changes               | Multiple (whenever `@Input()` changes) |

| `ngOnInit()`            | Initialize component data                    | Once (after `ngOnChanges()` on first load) |

| `ngDoCheck()`           | Custom change detection logic                 | Multiple (on every change detection cycle) |

| `ngAfterContentInit()`  | When projected content is inserted            | Once |

| `ngAfterContentChecked()` | After projected content is checked         | Multiple (after every change detection) |

| `ngAfterViewInit()`     | When view & child components are initialized  | Once |

| `ngAfterViewChecked()`  | After the component’s view is checked         | Multiple (after every change detection) |

| `ngOnDestroy()`         | Cleanup tasks before destruction              | Once (just before component is removed) |

  

What is "checking"? Is the process Angular does to check if the inputs of a component have changed.

  

So, in short this is how it works. When a component loads the first method is `ngOnChanges` because Angular needs to resolve inputs for rendering.

Then every method that ends in `init` only gets executed once. Think of this as ways of know that certain stages of the rendering process are completed.

Lastly, all the other remaining methods deal with changes happening in the DOM. If something changes Angular needs to propagate that change throughout the all DOM. So those methods are executed more than once, in fact the run with every change detection cycle.

  

(__note: `afterContentInit` comes before `afterViewInit`__)

  

`ngDoCheck` is a "special case".

  

`ngDoCheck` is meant to be used for change detection that Angular can't track. Because Angular tracks changes by object reference, if a property of an object changes, `ngOnChanges` won't pick that up. For example:

  

```typescript

@Component({

  changeDetection: OnPush

})

export class RandomOneComponent{

  @Input()

  public someobj: {counter: number} | null = null

  

  ngOnChanges(){

    /// does not pickup this.someobj.counter += 1!

  }

}

```

  

`ngDoCheck` is meant for scenarios like this but in general it's not meant to be used a lot.

  
  

## Change detection

  

Out of the box angular does `OnPush`. This will trigger changes all the time, regardless if you change a property or a reference. Basically you don't need to care about anything.

This comes at a performance cost because it's a lot of checking. Enter `OnPush`. With `OnPush` Angular only triggers change detection if a reference to an object changes. If you change a property nothing happens, unless you trigger it manually.

  

Does this impact signals? Yes. If you have an Input signal and set change detection to `OnPush` it behaves just the same way. Signals do not fix this "problem"

  

However if signals are available is signals all the way for. Disregard "old" inputs

  
  

# Angular Building Blocks

  

What are the main building blocks?

  

Modules - it's essentially a way of organizing related code together. It also helps the Angular compiler to resolve components and it to the DOM

  

Directives - are classes that add additional behavior to angular elements. There are 2 types of directives:

 - Attribute Directives - directives that change the behavior of elements (think highlight directive)

 - Structural Directives - directives that change the DOM layout. They always start with *ng

  

You can also think of components as a special type of directives that have a template.

  

Services - injectable classes used by DI

  

Pipes - classes responsible for data transformation

  
  

# Data binding

  

2 types: event binding and property binding.

  

One way data binding vs two way data binding: one way is "component changes view! and two way is "component changes view and view changes component" (banana in the box)

  

# Angular Modules

  

Purpose of modules -> organize relate things together. What things? Components and Providers

  

Declarations: directives that are part of this module. Can be components, pipes or custom  directives

Imports: what dependencies a module has

Export: what declarations this module exports

Providers: what services this module uses

Bootstrap: components you need during the bootstrap of the application. Like this:

  

```typescript

@NgModule({

  declarations:[],

  imports:[],

  exports:[],

  providers: [],

  bootstrap: [AppComponent, UserComponent]

})

export class AppModule

```

  

```html

<body>

  <app-root></app-root>

  <app-user></app-user>

</body>

```

  
  
  

Questions:

  

If you provide a service here, in which injector is it provided?

  

It depends: if the module is eagerly loaded, it's provided in the root injector. If the module is lazy loaded, providers are declared in a new injector created specifically for this lazy loaded module

  

Can you have multiple root modules?

  

Yes, you can have multiple root modules. Imagine a scenario here you have 2 angular apps running on the same page for example. If you do it like these 2 apps will be independent meaning they will have their own CD cycles and, even if one fails to start, the other will start

  

```typescript

import { bootstrapApplication } from '@angular/platform-browser';

import { appConfig } from './app/app.config';

import { AppComponent } from './app/app.component';

  

bootstrapApplication(AppComponent, appConfig)

  .catch((err) => console.error(err));

  

bootstrapApplication(SometOtherAppComponent, appConfig)

.catch((err) => console.error(err));

```

  

> By default all declarations within a module are private

  
  
  

```typescript

@NgModule({

  declarations:[],

  imports:[],

  exports:[],

  providers: [],

  bootstrap: []

})

export class AppModule

```

  

# Directives

  

Name some of the build in attribute directives: NgClass, NgStyle, RouterLink, NgModel

  

How do you pass data to Directives? Same as component, inputs. How do you get data out? Outputs

  

Which selectors can we use for directives?  Pretty much anything. It an be a string, a class name, a :not(), etc...

 ```typescript

 @Directive({

  selector: [app-highlight]

  selector: app-highlight

  selector: .highlight

  selector: :not(input[type=text])

 })

 ```

  

 How to listen to events? HostListener

  

 What Lifecycle hooks are available? All

  

 How to get ref to host element? in the ctor inject the elementRef

  

 Why is it bad to change element properties directly using the native element selector in a directive?

  

 1 - Decoupling? What if you don't run your app in the browser? Maybe `nativeElement` APi is not available. Using Angular abstractions is better

 2 - Performance? It's much more performant to let Angular deal with changes to the DOM rather than taking them out of the normal Angular flow

  

 Can you use ViewChild and ContentChild in directives? You can use contentChild() but not ViewChild() because Directives _do not have_ their own view. Same applies to ViewChildren and ContentChildren

  

 # Structural Directives

  

 What does the * mean in a directive? This asterisk is syntatic sugar for this:

  

 ```html

 <app-bananas *ngIf="someBool">

  

<!-- is the same as -->

  

 <ng-template [ngIf]="someBool">

  <app-bananas>

 </ng-template>

 ```

  

What happens if you attach an empty diretive to an element? The element disappears from the DOM. By default structural directives are lazy loaded meaning they are only loaded when you instruct Angular to do so (meaning you flip the ngIf).

  

# Angular Pipes

  

Pipe? A class responsible for data transformation

  

Built in pipes? date | json | uppercasePipe | Currency

  

What params are available to the transform method? Like this:

  

```html

<div>{{numberOfBytes | fileSize: "MB:GB" }}</div>

```

  

```typescript

transform(val: numberOfBytes, from: Mb, to:GB)

```

  

Pure vs Inpure pipe? Pure pipes execute the transform when the reference to the underling object is changed. Inpure pipes trigger every time CD runs

  

Pipe chaining? Yes, you can chain pipes

  

Pipes and performance: Inpure pipes run with every CD cycle which can cause performance issues. One way of improving the performance of your app is to rely more on pipes if you have a lot of functions in your templates (not click handlers, but instead functions that for example fetch properties from objects or something)

  

What is Async pipe? It's a built in pipe that allows you to subscribe and to unsubscribe automatically from observables

  

Does the pipe work only with observables? It handles observables and promises. Actually it works with everything that implements the `Subscribable<T>` interface.

  

Does async pipe trigger change detection? No. Async pipes `marksForCheck()` so the value updates in the next CD cycle but it does not trigger CD directly

  

Where should you async pipe? Everywhere where possible

  

# Angular Services

  

What is a service? Services are classes which handle business logic and share data between components

  

How? DI

  

How to provide services? `Inject` or `Providers:[]`

  

Why should you prefer `providedIn` syntax? Because it makes our services tree shakable

  

What values for `providedIn`? `root` meaning service is a singleton, `platform` will be injected at platform level which the parent for root injector; makes a singleton for _all_ applications on your page, `any` creates a separate instance of this service for every lazy-loaded module

  

Are services always singletons? Yes, _in the scope of the injector!!_

  

Can you inject services that are not marked as `@Injectable`? Yes but why would you?

  

# Directives Vs Components?

  

 - A Component is a directive which has a view. Actually `Component extends Directive`.

 - You can attach multiple directives to a DOM node (aka component) but only one component to a dom node

 - Intention! Directives modify existing component behavior, Components create views

  

 # Angular compilation - JIT VS AOT

  

 AOT - compile before running in browser

  

 Why do we need to compile? Because the browser can't understand angular things like components or directives

  

 JIT - transpile stuff at runtime, it requires the angular compiler to be loaded as well. Compiler is too big, half of the total size on Angular. Errors not caught until you run the project

  

 AOT - compile before run, no need to load compiler, we can catch errors at build time

  

 Which compilation phases in AOT?

  

 1 - Code analysis -> compiler goes through files and looks for angular specific stuff, mostly metadata collection

  

 2 - Resolve - resolve dependencies, make optimizations, etc...

  

 3 - template type checking  - validate expressions in our templates

  

 Template syntax limitations: https://angular.io/guide/aot-compiler#expression-syntax-limitations

  

 ## SSR

  

 What? Process of rendering the app on the server rather than the browser

  

 Why?

  1- Because it enables SEO. Crawlers cannot execute JS so they just get an empty HTML page which is bad for SEO. SSR prevents that.

  2- Improves first content paintful - good for page ranking

  3 - improved performance in old devices

  

  If it's so good why not use it all the time? No, some apps don't care about SEO. Think internal applications

  

  Pre-rendering static pages? this stragegy is when you pre-render some static pages during build time and gets kept in the bundle. Good for stuff that nevers changes

  

  # Lifecycle hooks

  

  what? Methods that are called automatically at some point in time by the angular framework

  

  name all 8 hooks? ngOnChanges, ngOnInit, ngAfterContentinit, ngAfterViewInit, ngOnDestryo, ngDoCheck, ngAfterViewChecked, ngAfterContentChecked

  

  What happens first? ngOnchanges if there are inputs, ngOnInit if no inputs

  

  What is executed before eery hook? The constructor

  

  Are lifecycle hooks available only for Directives and Components? Yes but pipes, service and modules also have `ngOnDestroy` for cleanup purposes

  

# Explain DI

  

"Classes should request dependencies from external sources rather than creating them"

  

What problem does this solve? DI solves the problem of coupling between classes, AKA, it makes our code less flexible and harder to test.

  

What parts are there to the DI pattern? Client - who asks for tokens; Injector - who keeps a list of tokens and decides which token to use; service - the actual implementation of the tokens

  

# What makes DI in Angular so important

  

## On the module/ providers side of things

  

Because it has a hierarchical structure. What is that?

  

Angular has a root injector. If we want to provide services here we use `providedIn: root` or register them in the Providers array of the module (if there are NOT lazy loaded)

  

Root Injector is a child of platform injector. This is called when we invoke `platformBrowserDynamic`

  

But there is yet another injector! The Null injector which is a parent of Platform Injector. Funny enough you cannot register any services there. It just throws errors if angular cannot

solve dependencies elsewhere in the other injectors

  

You can also create your own child injectors if you lazy load modules because a lazy loaded module has it's own injector.

  

## On the component/directive side of things

  

You can also have Node Injectors if you inject a service in a Directive like this

  

```typescript

@Component({

  Providers: [SomeService] //this will be at NodeInjector level

})

public class SomeComponent(){}

```

  

However if you start rendering child components inside `SomeComponent` they will also create their own injector. Think of this like parent -> child -> grandChild

  

## How does Angular resolve dependencies?

  

It goes up from the most specific to the least.

  

Imagine you have a grandChild -> child -> parent structure. Angular looks in every injector from grandChild up to parent to find the token you asked for. If it doesn't find it, it

then searches in the module injectors (so root -> platform). If it reaches the Null injector you blow up with "no provider for bla bla bla" error.

  

What happens if one of the modules is lazy loaded?

  

It's the same thing BUT instead of going from node injectors directly to root injector it first looks in the injector created by the lazy loaded module.

  

## Resolution modifiers - aka how to hijack the flow above

  

Why? Give you control over how to resolve the dependencies

  

How? In the constructor

  

@Self - only look inside this local injector and throw if nothing there

  

@SkipSelf - skip the local injector and start from the parent injector

  

@Optional - tells angular this shouldn't blow up if no dependency is found

  

@Host - bit weird but this means in the same component view. Think a split form in different components in which you want need share the formGroup. `Host` means "I only care about the

formGroup of my host component and that's it. Blow up otherwise"

  

Also you can combine multiple modifiers at the same time - @Self() @Optional

  

```typescript

export class Bananas(){

  constructor(@Self() userService: userService){

  

  }

}

```

  

## Dependency Providers

  

What? Give you control on how the dependencies are created

  

How?

  

```typescript

useClass: provide a new instance

useExisting: find an existing instance, fail if no class exists

useFactory: useful when you want to do additional logic before instantiating a provider

useValue: provided non class object. Think window object and running angular outside of a browser context

```

# Injection Tokens

  

What? A special class that allows to inject numbers, strings and other shapeless objects like arrays, functions ,etc...

  

Remember! You cannot inject primitives!

  

```typescript

export class Bananas(){

  constructor(private url: string){ // this does not work!

  

  }

}

```

  

Can you configure our injections tokens? Yes

  

```typescript

export const endpoint = new InjectionToken<string>('url', {

  providedIn: 'root'

  factory: () => 'bananas

})

```

  
  

How do we inject the tokens? Almost like class based injection

  
  

```typescript

export class Bananas(){

  constructor(@Inject(MyToken) myToken){}

}

```

  

Also note that the above is the "real" way of injecting stuff. You can inject anything like this. The usual `private readonly userService: UserService` is the same as this

  

# Angular Ivy

  

What is Ivy? A new compiler. It requires AOT compilation

  

What is NGCC? Is Angular Compatibility Compiler. Basically make old code usable in Ivy

  

Benefits of Ivy:

- Improves rebuild time

- Improves tree shaking

- Improves strict type checking

  
  

# NgRx

  

## If using "normal" store

  

4 components you need to remember: action, selectors, reducers and effects

  

Actions - define what you can do with your store. This is the stuff you can dispatch

Reducers - where you define your state and how your state changes

Selectors - how you fetch state from your store

Effects - Side effects that allow your main state to focus only on managing external data. Think loaders or logging operations or something like that

  

Flow is as follows: action is dispatched -> reducer is invoked -> state changes -> selector is invoked -> state is fetched -> effect is invoked if defined

  

You can have multiple stores in your application but you always have a top level store, the `root` store. You can then have stores at feature level

  

## Setup

  

Setup is key here, it's either done the way it needs to be or it doesn't work. NgRx offers some stuff that allows you to skip some configuration parts but I find it's easier to just do the boilerplate. Also it's best to keep "one reducer for one piece of state".

  

### Setup if using standalone components

  

- Actions

```typescript

enum SomeActions {

  LOAD_BANANAS = '[BANANAS] LOAD',

  DELETE_BANANAS = '[BANANAS] DELETE',

}

  

export const loadBananas = createAction(SomeActions.LOAD_BANANAS)

export const deleteBananas = createAction(SomeActions.DELETE_BANANAS, props<{id: string}>())

```

  

- Reducers

  

```typescript

export bananasFeatureKey = 'bananasFeatureKey'

  

export interface BananasState {

  bananas: string[]

}

  

const initialState = []

  

export const bananasReducer = createReducer(initialState,

  on(loadBananas, (state,action) => {

    return [...state, ...action]

  })

)

```

  

- Selectors

  

```typescript

const bananasSelector = createFeatureSelector<BananasState>(bananasFeatureKey) //this is critical!!!

  

export const selectLastBanana = createSelector(bananasSelector, state => state[state.length-1])

```

  

- If your state is for `root`, do this is `app.config.ts`

```typescript

export const appConfig: ApplicationConfig = {

  providers: [provideZoneChangeDetection({ eventCoalescing: true }),

  provideRouter(routes),

  provideStore(),

  provideState(appStateFeatureKey, appStateReducer)]

};

```

  

- If your state is for `feature`, do this in `app.routes`

```typescript

export const routes: Routes = [

    {

        path: 'some-feature', component: SomeFeatureComponent,

        providers: [provideState(someFeatureStateKey, SomeFeatureStateReducer), provideEffects(SomeFeatureEffects)]

    }

];

```

  

- If you need to register effects

  

Remember an Effect is an injectable. You need to register the effects as well. See above

  

```typescript

import { Actions, createEffect, ofType } from '@ngrx/effects'

  

@Injectable()

export class BananasEffects{

  private actions$ = inject(Actions) //this comes from ngrx, it's not our actions

  

  logToDatabaseOnBananaLoaded$ = createEffect(() => {

    return this.actions$

    .pipe(

      ofType(SomeActions.LOAD_BANANAS),

      switchMap(() => {

        return this.dbService.log('bananas loaded at 11:11')

      }),

      switchMap(() => {

        return of({type: SomeActions.BANANAS_LOADED_SUCCESS})

      })

    )

  })

}

```

  

Effects needs to return an action if you don't specify `{dispatch: false}` in the options (the second argument to `createEffect` after the effect definition)

  
  

### Setup if using modules

  

All the action, reducer, selector stuff is the same as before. It just changes the registration syntax slightly in the modules

  

```typescript

@NgModule({

  declarations: [

    AppComponent

  ],

  imports: [

    BrowserModule,

    AppRoutingModule

  ],

  providers: [StoreModule.forRoot({appState: AppStateReducer})],

  bootstrap: [AppComponent]

})

export class AppModule { }

```

  

If it is a feature module...

  

```typescript

@NgModule({

  declarations: [

    AppComponent

  ],

  imports: [

    BrowserModule,

    AppRoutingModule

  ],

  providers: [StoreModule.forFeature({featureState: featureStateReducer})],

  bootstrap: [AppComponent]

})

export class AppModule { }

```

# Change Detection

  

## What is Change Detection

  

It's the mechanism is responsible for synchronization between view and component

  

## What triggers CD?

  

 - Events: click, keydown, mouseenter, etc as long as the event has an handler registered

 - SetInterval and SetTimeout

 - When an AJAX call or promise is being resolved

  

## How does Angular track that?

  

Using zone.js

  

## What is zone.js?

  

A library that creates an execution context for an Angular App and helps to detect async events & run CD cycles by monkey-patching the native browser API

  

## WTF you mean by monkey-patching?

  

This means extending the original behavior of native browser APIs but also add something else. For example, in Chrome Dev Tools you can do:

  

```typescript

const defaultAlert = window.alert

  

window.alert = function(str){

  console.log('before alert')

  default.apply(this, 'bananas')

  console.log('after alert')

}

```

  

This would output: `before alert` -> `bananas` as an alert dialog -> `after alert`. And this way you just extend the original behavior of alert. ZoneJs does something similar but for API like timeouts.

  

## High level overview of CD cycle

  

![changeDetection](./cd_1.png)

  

What you need to remember where is that it's top to bottom. Zone js creates a zone listening for browser events from the root of your app all the way to the last component.

  

## On Push Change Detection

  

Means telling Angular not to check this components for changes in every CD cycle unless:

 - an @input has changed it's _reference_ (not value)

 - an event has been fired on this component (think click event here)

 - you exactly marked the component for changes (markForCheck())

  

You also need to remember that marking a component for changes also parses the full component tree

  

Can you detach a competent from change detection altogether? Yes, `this.cd.detach()`. You need to then do all CD yourself

  

### Explain runOutsideAngular

  

It's a way of doing async stuff of handling events without impacting change detection. It's essentially a way of saying "treat this not as Angular code"

  

# Forms

  

## Template vs Reactive

  

Template -> you handle the form via a set of special directives -> sync and mutable (meaning you are actually changing properties in the component via [(ngModel)])

Reactive -> gives you explicit access to the Form model -> async and immutable (as in there is no property of the component you are mutating except the form)

  

When to use which? reactive all the time, template if you have a super small form (like one input)

  

### Building blocks

 - FormControl - represents one HTML control

 - FormGroup - a group of FormControls

 - FormArray - aggregate controls as an array instead of an object

  

### Explain what is NgModel and Control Value Accessor

  

- 2 way data binding, banana in the box, binding and event all in one

  

- CVA: an abstraction that allows you to turn anything into a form control

  

### Form validators

  

Reactive -> just a function. You can define your own validators and pass it to the formControl

Template -> directives. If you want to create your own custom validator you need to create a directive and attach it to the formControl manually

  

Async Validators -> same thing but you return an `Observable` instead of `null/error`

  

```typescript

 private someCustomValidator: ValidatorFn = (val: AbstractControl) => {

    if(val) return null

    else return {error: 'Bananas'}

  }

  

  protected b = new FormControl('', {validators: [this.someCustomValidator]})

```

  

How do you do cross-control validation? Same structure for `ValidatorFn` but you need to pass the validator option at form group level

  

```typescript

protected crossValidator: ValidatorFn = (controls: AbstractControl) => {

    const name = controls.get('name')?.value

    const surname = controls.get('surname')?.value

  

    if(name+surname === 'bananas'){

      return null

    } else {

      return {error: 'not bananas'}

    }

  }

  

  protected fb = inject(NonNullableFormBuilder);

  

  protected myForm = this.fb.group({

    name: new FormControl('', Validators.required),

    surname: new FormControl('', Validators.required)

  }, {

    validators: [this.crossValidator]

  })

  ```

  

# Standalone Components

  

## What is a standalone components?

  

A component that is declared without an NgModule, thus making NgModules optional

  

__note: if you have module that imports a standalone component, that component is placed in the `imports` array and NOT the `declarations`__

  

## Why?

  

To make the component the smallets building block thus making the framework easier

  

## What other standalone stuff is there?

  

Directives and Pipes

  

## Bootstrap application

  

If in standalone world the function is now:

  

```typescript

//main.ts

bootstrapApplication(AppComponent, appConfig) //App component needs to be a standalone component

  .catch((err) => console.error(err));

```

  

Other useful stuff:

  

- Interop between modules and standalone config for providers

  

```typescript

//app.config.ts

  

export const appConfig: ApplicationConfig = {

  providers: [provideZoneChangeDetection({ eventCoalescing: true }, importProvidersFrom(RouterModule.forRoot(routes))),

  provideRouter(routes),

  provideHttpClient(),

  provideMarkdown(),

  provideAnimationsAsync(),

  provideStore({ atlas: atlasReducer })]

};

```

  

## How does it work under the hood and what restrictions there are?

  

It's a virtual SCAM pattern (singular component angular model). It's not possible to bootstrap multiple root components in the same page.

  

# Dom Manipulation - View, ViewRef, ViewContainerRef, EmbeddedView

  

 ## What is a view?

  

> The smallest unit of the UI that represents a portion of the screen rendered

  

## What type of views are there?

  

3 types: component view, embedded view and host view.

  

### Component view

A component view is, as the name implies, the view of the component (think of this like your html file)

  
  

### Embedded View

The embedded view is every view that does not belong the component directly. What do I meant by that?

  

```html

<div>

  <h1>hello</h1>

  <p>bananas</p>

<ng-template #myTemplate>

  <!-- this is the embedded view. Whatever you give throw at it here it's NOT part of the view of the component -->

  <ng-template/>

</div>

```

  

*note: every time you use `ng-template` creates an embedded view*

  

### Host View

A view that is created when programmatically adding a component (via `ViewContainerRef.createComponent()`)

  

## How does Angular manages views?

  

2 mechanisms: `ViewRef` and `ViewContainerRef`. The difference between the two is that `ViewRef` holds only a single view (one of the three we talked above), whereas `ViewContainerRef` is a container of many views.

  

### But what the hell is a ViewContainerRef

  

"A container of many views". How come a container have many views? And remember view here is "the smallest piece of UI", don't think of view in the sense of component view.

  

The only for a container to have many views is if it has multiple elements. So would the following be a VCR?

  

```html

<ul>

    <li>bananas</li>

    <li>apples</li>

    <li>pears</li>

</ul>

```

  

Even though it has multiple "views" the answer is no. Why? Because all these elements are static. Angular *knows* exactly what to render here. It knows that this piece of markup is not going to change ever. But what if we had the following?

  

```html

<ul *ngIf="displayFruits">

    <li>bananas</li>

    <li>apples</li>

    <li>pears</li>

</ul>

```

  

In this case Angular does create a VCR associated with the `ul` element. Why? Because it needs to dynamically be able to hide/show the list if the `displayFruits` boolean changes.

  

So it looks like stuff that can change dynamically can create VCRs... And that is exactly right!

  

The elements that create a VCR in Angular are structural directive anchors (*ngIf, *ngFor, ng-template)... and components! Yep, your component also creates a VCR.

  

So in a component like this...

  

```html

<ul *ngIf="displayFruits">

    <li>bananas</li>

    <li>apples</li>

    <li>pears</li>

</ul>

```

  

... you would have 2 VCRs

  

*note: in the example above if you removed the ngIf and used a view child query like this - `viewChild.required('list', { read: ViewContainerRef })` you still get back a view container reference even though there isn't one there. Angular just wraps the query in a VCR for your for consistency sake*

  

What can we do with VCRs?

  
  
  

### On to ViewRefs...

  

This is where things get weird. Accodring to angular, a view ref is somethat that "Represents an Angular view." 🤷‍♂️

  

So one would assume "oh I can just go and fetch the viewRef for my current compnent"...Maybe you try something like:

  

```typescript

protect myViewRef = inject(ViewRef)

```

  

Angular will complain with a null provider exception. you think "ok this is unexpected...maybe if you go via the VCR?"

  

```typescript

this.vcr.get(0)

```

  

You get `null` back. You try...

  

```typescript

this.vcr.element

```

..and you get back an `ElementRef`. You proceed to looking at the remaining methods and nothing there seems to say "get this view".

  

So how???

  

You don't. And this is what confused me. You don't directly access a ViewRef, you need to *create it*. The only way for you to get a ViewRef is by telling angular "go and create a view in this vcr"

  

You can do this in 2 ways:

  

```typescript

    const d = this.list().createComponent(SimpleComponent) //gives you a ComponentRef. You can then reference the host property to get the ViewRef

    const c = this.list().createEmbeddedView(this.someTemplate()) //Gives you back a viewRef

  ```

  
  
  
  
  
  
  
  
  
  
  
  
  

##### bananas

CVA

cross field validation

inject components dynamically - Template Ref vs Element ref

multi: true

hot cold observables

  
  

---

Ongoing: angular/ts/jest

  

Study:

performance

browser API

CSS/SASS/Layouts

acessability