---
layout: post
title: Subclass of Array .slice() calls constructor() with length
---

I guess this is a bug, I haven't been able to find anything else about this. If I do this:

```javascript
class MyArray extends Array {

    constructor(...i) {
        super();
        console.log(JSON.stringify(arguments));
    }

}

const my = new MyArray(88, "cat");
my.push("what");
my.push("when");
my.push("why");
my.push("how");

my.slice();
```

I get:

```bash
{"0":88,"1":"cat"}
{"0":4}
```

Notice that the length of the array is passed to the constructor as the first argument when it is sliced.

This also applies to splice().
