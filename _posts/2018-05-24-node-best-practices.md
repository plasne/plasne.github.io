---
layout: post
title: My Node.js Best Practices
---

# My Node.js Best Practices

My preferred development language is Node.js. Recently I have been doing a number of projects with other folks and it becomes beneficial to standardize on some best practices. This is by no means exhaustive, but it an attempt to write down some of the practices I prefer to use.

## Classes

In ECMA2015, Classes were added to JavaScript, while just a convenience over prototypes, this does make things must cleaner and more readable.

Below is the standard format showing a constructor, a property, and a method:

```javascript
class Test {

  get name() {
    // return something
  }
  set name(value) {
    // set something
  }

  doWork() {
    // do some work
  }

  constructor(value) {
    // initialize stuff
  }

}
```

You can export it and instantiate it like this:

```javascript
module.export = class Test {
  // my properties and methods
}

const test = new Test();
```

You can inherit like this:

```javascript
class Base {
  get hello() {
    return "hello";
  }
  constructor(value) {
    // initialize stuff
  }
}

class Derived extends Base {
  constructor(value) {
    super(value);
  }
}

const derived = new Derived();
console.log(derived.hello);
```

## Private Variables

## Promises

## Async / Await

## Template String Literals

## Scope Variables Properly

## Commander

## Connection Pooling

## Winston

## Configuration

## Axios vs Request 
