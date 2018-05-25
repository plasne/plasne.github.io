---
layout: post
title: My Node.js Best Practices
---

My preferred development language is Node.js. Recently I have been doing a number of projects with other folks and it becomes beneficial to standardize on some best practices. This is by no means exhaustive, but it an attempt to write down some of the practices I prefer to use.

This post talks about some new features in ES6, 7, and 8. This external post goes into much greater detail on new features:
[https://medium.freecodecamp.org/here-are-examples-of-everything-new-in-ecmascript-2016-2017-and-2018-d52fa3b5a70e](https://medium.freecodecamp.org/here-are-examples-of-everything-new-in-ecmascript-2016-2017-and-2018-d52fa3b5a70e)

## Classes

In ES6, classes were added to JavaScript, while just a convenience over prototypes, this does make things much cleaner and more readable. I see a lot of people still creating objects using functions, closures, factories, etc. and there are good reasons for that sometimes, but for your standard objects, I like classes.

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

You can export it:

```javascript
module.export = class Test {
  // my properties and methods
}
```

And then instantiate it like this:

```javascript
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
    // Derived will inherit the property "hello"
  }
}
```

## Private Variables

There are quite a few ways to handle "private" variables in objects now, but I like the following use of WeakMaps (added in ES6):

```javascript
const _myValue = new WeakMap();

module.exports = class MyClass {

  get myValue() {
    return _myValue.get(this);
  }
  set myValue(value) {
    _myValue.set(this, value)
  }

}
```

The variable remains private because it wasn't exported and each object can have it's own values by specifying itself as the key.

```javascript
const myObject1 = new MyClass();
myObject1.myValue = "red";

const myObject2 = new MyClass();
myObject2.myValue = "blue";

console.log(myObject1.myValue); // red
console.log(myObject2.myValue); // blue
```

## Promises

Promises were added native to the language in ES6, so there is no reason not to use them everywhere in place of callbacks. Node now includes a "util.promisify" function and a "util.promisify.custom" symbol to convert functions with callbacks into promises.

There are still plenty of reasons to use other promise libraries. [Bluebird](http://bluebirdjs.com/), for instance, offers a lot of additional functionality. Also, with any promise implementation, you can start to stack things on top of it... check out npm for lots of concurrency, queues, etc. libraries that work with promises.

Personally I often just implement a promise around a callback function because there may be other cases I want to handle, like in this example, where I want an error thrown for an unsuccessful HTTP call:

```javascript
const request = require("request");

function query(url) {
  return new Promise((resolve, reject) => {
    request.get(url, (error, response, body) => {
      if (!error && response.statusCode >= 200 && response.statusCode < 300) {
        resolve(body);
      } else if (error) {
        reject(error);
      } else {
        reject(new Error(`${response.statusCode}: ${response.statusMessage}`));
      }
    });
  });
}
```

Rather than messy callback after callback, you can start to implement promise chains:

```javascript
query(url)
.then(body => {
  return JSON.parse(body);
})
.then(json => {
  // do something with the JSON
})
.catch(ex => {
  // handle error
});
```

There are tons of other things you can do, but one of my favorite is waiting on a bunch of work to finish:

```javascript
const promises = [];

promises.push( query("http://some_url") );
promises.push( query("http://some_other_url") );

Promises.all(promises)
.then(results => {
  // results is an array of the responses [ body1, body2 ]
})
.catch(ex => {
  // handle error
});
```

Make sure you always end your outer-most promise chain with a .catch() or you will get an unhandled promise rejection error message and they can be problematic to track down.

## Async / Await

Promises solve the messy nested callbacks, but when you have a lot of them, you can end up with a bunch of nested promises and it doesn't improve the readability that much. Aysnc/await (ES8) can make this much more legible:

```javascript
async function myFunction() {
  const result = await myFunctionThatReturnsPromise1();
  await myFunctionThatReturnsPromise2(result);
}

myFunction
.then(() => {
  // do stuff after function is complete
})
.catch(ex => {
  // handle error
});
```

Notice that the async function is itself a promise.

## Arrow Functions

Arrow Functions (ES6) make a lot of things easier to write and read (I have used them through-out this article), but a few things to keep in mind, the function:

* doesn't have its own "this"
* doesn't have "arguments", but it can use rest parameters
* doesn't support "new"
* cannot yield (ie. cannot be used as a generator)

One thing in particular I am fond of is Immediately Invoked Function Expressions (IIFE), where the function exists just to run immediately. Consider the above async/await example where the invokation of myFunction was still a promise, it might look better like this where async and a try/catch can be used:

```javascript
async function myFunction() {
  const result = await myFunctionThatReturnsPromise1();
  await myFunctionThatReturnsPromise2(result);
}

(async () => {
  try {
    await myFunction();
  } catch (ex) {
    // handle errors
  }
})();
```

Or what about this example:

```javascript
// without-IIFE, with a hex variable that will be set based on the switch, ugly!
switch (color) {
  case "red":
    hex = "#FF0000";
    break;
  case "green":
    hex = "#00FF00";
    break;
  case "blue":
    hex = "#0000FF";
    break;
}

// instead, this is pretty elegant
const hex = ((_color) => {
  switch (_color) {
    case "red":   return "#FF0000";
    case "green": return "#00FF00";
    case "blue":  return "#0000FF";
  }
})(color);

// this is perhaps even better, but not an example of a IIFE
const colors = { "red": "#FF0000", "green": "#00FF00", "blue": "#0000FF" };
const hex = colors[color];
```

IIFE can also help with scoping variables, consider:

```javascript
const color = "red";

(() => {
  // I need a color in here too, but didn't pay attention to that already existing
  const color = "green";
  console.log(color); // green in here
})();

console.log(color); // it's ok, it's still red
```

## of vs in

Much is written on the difference between "of" (ES6) and "in", however, as difficult as it is to remember, when working with things that iterable, use "of" and when wanting the members of an object, use "in".

This is made a bit difficult when context switching as most other languages are for...in when dealing with collections, but Node.js is for...of.

## Template String Literals

Template String Literals (ES6) make it a lot easier and cleaner to build strings. Consider the following example:

```javascript
const without = "1. " + name + ": \"" + (value || 0) + "\"";
const with = `1. ${name}: "${(value || 0)}"`;
```

## Scope Variables Properly

There seems to be a habit with a lot of folks to use "var", but in pretty much every case "const" or "let" would be better. Both "const" and "let" are block-scoped which goes a long way towards making sure you don't set a function-scoped "var" to something because you didn't remember you used that variable name already.

I use "const" for almost everything (as you have probably noticed already in this article), something like a simple counter or total are about the only things I use "let" for because they logically need to change. Otherwise, if I am doing something like a multi-step operation I might declare multiple variables using const which makes it more readable.

```javascript
const distance = (trip1 + trip2 + trip3);
const mileage = (distance * 0.30);
```

Keep in mind that "const" only prevents you from re-assigning the variable:

```javascript
const colors = [];
colors.push("red", "green"); // ok
colors = ["red", "green"];   // exception

const sizes = { big: 10, med: 5, small: 2 };
sizes.med = 7;                         // ok
sizes = { big: 10, med: 7, small: 2 }; // exception
```

## Connection Pooling

There are dozens of ways to host applications today. Consider Azure for a moment, you could host:

* in a Function in a consumption App Service Plan
* in a Function in a dedicated App Service Plan
* in a Web App in a consumption App Service Plan
* in a Web App in a dedicated App Service Plan
* in a Docker container running in Azure Container Instance
* in a Docker container running in Azure Container Service
* in Service Fabric
* in a Virtual Machine exposed via a Load Balancer
* in a Virtual Machine with a public IP
* and so on...

All of those solutions have different constraints on the number of outbound connections that are allowed due to SNAT, overlay networks, etc. To make your application portable across multiple hosting methods you need some way to constrain the outbound ports. The 2 things you want to do is create an agent with a pool of sockets that won't go beyond a maximum and keep-alive. The pool will ensure you don't have too many sockets while the keep-alive setting will keep the sockets open for a while so they can be reused, significantly improving performance.

## Commander

[Commander](https://github.com/tj/commander.js) is absolutely one of my favorite packages for Node.js. It gives you command line options, actions, help, etc, and is easy to use. Almost every application I write has some use for Commander.

## Winston

[Winston](https://github.com/winstonjs/winston) is a full-featured logger framework, if you need to do more than console.log and console.error, this is a great library to look at. I tend to like the color coding for event levels and to keep all event levels the same size, so you can do something like this:

```javascript
const logColors = {
    "error":   "\x1b[31m", // red
    "warn":    "\x1b[33m", // yellow
    "info":    "",         // white
    "verbose": "\x1b[32m", // green
    "debug":   "\x1b[32m", // green
    "silly":   "\x1b[32m"  // green
};
const logger = winston.createLogger({
    level: "debug",
    transports: [
        new winston.transports.Console({
            format: winston.format.combine(
                winston.format.timestamp(),
                winston.format.printf(event => {
                    const color = logColors[event.level] || "";
                    const level = event.level.padStart(7);
                    return `${event.timestamp} ${color}${level}\x1b[0m: ${event.message}`;
                })
            )
        })
    ]
});
```

## Configuration

There are a myriad of ways to handle configuration settled on using Environment Variables because that seems to be a widely accepted method and because it makes it easy to control the settings across environments. However, I still like the idea of allowing for the variables in a file, so [dotenv](https://github.com/motdotla/dotenv) addresses that, and I still like the idea of specifying command line parameters using Commander.

Below is what I will use as an implementation standard whereby settings are determined in the following priority order:

1. Command line
2. Environment variable
3. Environment variable from file
4. Default

```javascript
const logLevel   = cmd.logLevel   || process.env.LOG_LEVEL           || "error";
const port       = cmd.port       || process.env.PORT                || 8080;
```
