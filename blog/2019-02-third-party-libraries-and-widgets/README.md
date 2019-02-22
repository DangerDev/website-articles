---
title: "How to integreate third party libraries and widgets into Angular"
author: Johannes Hoppe
mail: johannes.hoppe@haushoppe-its.de
published: 2019-02-21
keywords:
  - Angular
  - jQuery
  - library
  - libraries
language: en
thumbnail: ../angular.png
hidden: true
---

**
For an upcoming workshop we were asked how to integrate third-party libraries into an Angular application.
In this blog post we want to have a closer look at this question.
We will discuss a few possible strategies and weigh them against each other.**

> Note: The whole article is based on the assumption that you are using the [Angular Cli](https://cli.angular.io/).

<hr>

Table of contents:

* [General Considerations](/blog/2019-02-third-party-libraries-and-widgets#general-considerations)
* [Integrating a pure ES6 JavaScript Library](/blog/2019-02-third-party-libraries-and-widgets#integrating-a-pure-es6-javascript-library) (lodash.clonedeep)
* [Integrating a JavaScript Widget](/blog/2019-02-third-party-libraries-and-widgets#integrating-a-javascript-widget) (plotly.js)
* [Integrating a jQuery Widget](/blog/2019-02-third-party-libraries-and-widgets#integrating-a-jquery-widget)


## General Considerations
   
First of all, we would like to state that it is basically a better idea to use native Angular modules.
This is the only way to profit from an optimal bundle size.
However, it is often simply necessary to fall back on existing solutions that already meet all technical requirements and thus save a lot of time and money.

Usually the following questions should be answered in advance in order to keep the effort as low as possible:

- Are there alternatives that are already based on Angular and how much would be the effort to use this alternative?
- Is the library / widget compatible with ES2015 (ES6) Modules or do we have to use the global object (`window`)? 
- How big is the foreign code? Will it slow down the build process significantly. Can we use a CDN if necessary?
- Is jQuery a dependency? (jQuery itself is also quite large, see [jQuery file size](https://mathiasbynens.be/demo/jquery-size))



## Integrating a pure ES6 JavaScript Library

As an example of a perfect third party library we would like to introduce lodash.
Lodash is the Swiss Army Knife for all kinds of programming tasks.
It is well organized and supports ES2015 modules.

For example, if we want to make a deep copy of an object, we are very well served with Lodash.
First, we have to install it:

```bash
npm install lodash
npm install @types/lodash --save-dev
``` 

Now we are able to import the method as usual.
The command looks like this:

```ts
import { cloneDeep } from 'lodash';

const nestedObject = {
  nested: {
     hello: 'world'
  }
}
const deepCopy = cloneDeep(nestedObject);

nestedObject === deepCopy // false
nestedObject.nested === deepCopy.nested // false (it's a deep copy)
```

Since clean tree shaking [can be tricky](https://medium.com/@martin_hotell/tree-shake-lodash-with-webpack-jest-and-typescript-2734fa13b5cd), we can also try a separate package.
So we don't install the whole codebase but only the needed part and its types.

```bash
npm install lodash.clonedeep
npm install @types/lodash.clonedeep --save-dev
```

```ts
import cloneDeep from 'lodash.clonedeep';
```

This doesn't just apply to lodash.
We should always check how big the bundles will be by our new dependencies.
In fact, if we work with 3rd party libraries, the bundle sizes will become one of the biggest showstoppers.

**[👉 Demo on Stackblitz](https://stackblitz.com/edit/angular-3rd-party-libraries-and-widgets?file=src%2Fapp%2Flodash-example%2Flodash-example.component.ts)**


# Integrating a JavaScript Widget

Lets take a look at [plotly.js](https://plot.ly/javascript/).
It's is a high-level, declarative charting library, which is built on top of [d3.js](http://d3js.org/) and [stack.gl](http://stack.gl/),
The library ships with many preconfigured chart types, including scientific charts, 3D graphs, statistical charts, SVG maps, financial charts, and more.

![plotly.js demo](plotly_2017.png)

We can again start by installing it via npm

```bash
npm install plotly.js-dist
```

and import plotly.js as

```ts
import Plotly from 'plotly.js-dist'; 
```

But beware, plotly.js with all it's depedencies (including D3.js) is huge!
Again, we can save a lot of bundle size if we choose the right package.
Please refer to the official [bundle information](https://github.com/plotly/plotly.js/blob/master/dist/README.md#bundle-information) to choose the right partial bundle.

The main plotly.js bundle weights in at:

| plotly.js | plotly.min.js | plotly.min.js + gzip |
|-----------|---------------|----------------------|
| 6.1 MB    | 2.8 MB        | 849.5 kB             |

That's a hell of a lot of code to draw a pie chart, for example.
If we just want to draw a pie chart we can choose the `basic` partial bundle instead.
It contains trace modules `scatter`, `bar` and `pie`:

| Raw size | Minified size | Minified + gzip size |
|----------|---------------|----------------------|
| 2.3 MB   | 810.9 kB      | 264.8 kB             |

At least that's a little better.
<!--
There are also CDN links available, which can be used, too.
We will take a look at them later on.
https://cdn.plot.ly/plotly-basic-1.44.4.min.js
-->

So we are going to install [`plotly.js-basic-dist`](https://www.npmjs.com/package/plotly.js-basic-dist) vias

```bash
npm install plotly.js-basic-dist
```

and import it like this:

```ts
import Plotly from 'plotly.js-basic-dist'
```

The plotly.js packages with the `-dist` suffix contain a ready-to-use plotly.js distributed bundle.
It is not minified, but we don't want it to be minified here.
Instead, we will minify the code from plotly.js along with the other code in the productive build of Angular (`ng build --prod`).
It's generally not a good idea to minify something twice!

Ok. Let's start.

The idea behind plotly.js is quite simple.
We have a `<div>` element, get a reference to it and draw a "plot" into it. 
In a world without Angular our code would look like this:

```ts
const myDiv = document.getElementById('id_of_the_div')

const data = [{
  values: [66, 22, 12],
  labels: ['Angular', 'React', 'Vue'],
  type: 'pie'
}];

const layout = {
  title: 'Top 3 Most Popular SPA Frameworks in 2019*',
  height: 400,
  width: 500
};

Plotly.newPlot(myDiv, data, layout);
``` 

<small>(&ast;Note: exactly one person was interviewed for this ranking.)</small>

The question is, where do we get a reference to the DOM element from?

The simplest way provided by angular is the wrapper `ElementRef`.
It's the return type of the `@ViewChild()` decorator, if there is no component applied (otherwise it will return a component instance instead). 
The decorator accepts various selectors, as described [here](https://angular.io/api/core/ViewChild). 
We will use a template reference variable as a string, so `@ViewChild('myDiv')` will query against `<div #myDiv></div>`.
The `ElementRef` will be ready when `ngAfterViewInit()` is called:

```ts
import { Component, ViewChild, AfterViewInit, ElementRef } from '@angular/core';
import Plotly from 'plotly.js-basic-dist'

@Component({
  selector: 'app-plotlyjs-example',
  templateUrl: './plotlyjs-example.component.html',
  styleUrls: ['./plotlyjs-example.component.css']
})
export class PlotlyjsExampleComponent implements AfterViewInit {

  @ViewChild('myDiv')
  myDiv: ElementRef;

  ngAfterViewInit() {

    const myDivEl = this.myDiv.nativeElement;

    var data = [{
      values: [66, 22, 12],
      labels: ['Angular', 'React', 'Vue'],
      type: 'pie'
    }];

    var layout = {
      title: 'Top 3 Most Popular SPA Frameworks in 2019*',
      height: 400,
      width: 500
    };

    Plotly.newPlot(myDivEl, data, layout);
  }
}
```

There are multiple other ways to get a reference to a DOM element.
We recommend the following article if you are interested in other ways to get a reference to a DOM element: [Angular in Depth: Exploring Angular DOM manipulation techniques](https://blog.angularindepth.com/exploring-angular-dom-abstractions-80b3ebcfc02)

**[👉 Demo on Stackblitz](https://stackblitz.com/edit/angular-3rd-party-libraries-and-widgets?file=src%2Fapp%2Fplotlyjs-example%2Fplotlyjs-example.component.ts)**


# Integrating old jQuery Widgets

As we have seen, ES2015 modules are an ideal way to use third-party libraries.
However, not all third-party libraries support this modern way.
These libraries assume that jQuery is available in the global scope and have no clue about modules.


Fortunately, the Angular CLI provides a declarative way to provide these libraries/widgets via the `angular.json` file.
First we install jQuery and the library with the help of npm.
It is important to pay attention to the version of jQuery.
Not all libraries support jQuery v3, which has a some of breaking changes.

```bash
npm install jquery@3.3.1
npm install @types/jquery@3.3.29 --save-dev
```






Of course this is not great, but you can't do anything about it.
Your boss still wants this scheduler today and the widget should look "like in Outlook".
That's why you need to get this pro license for [Kendo UI for jQuery](https://www.telerik.com/kendo-jquery-ui), because you have to deliver! 😎



Now we will install the full version of Kendo UI for jQuery.
Please keep in mind that the vendor provides customized versions, too.

```bash
npm install @progress/kendo-ui
```

The Angular CLI provides an entry in the `angular.json` file for such global scripts.