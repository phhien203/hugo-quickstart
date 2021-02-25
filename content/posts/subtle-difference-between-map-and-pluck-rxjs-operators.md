+++
title = "Subtle difference between map and pluck RxJS operators that you should know"
date = "2021-02-24T09:18:08+07:00"
author = "Hien Pham"
authorTwitter = "HienHuuPham" #do not include @
cover = ""
tags = ["rxjs", "angular"]
keywords = ["map", "pluck"]
description = "Explanation in detail about how map and pluck work, and then figuring out main difference between two operators"
showFullContent = false
+++
Do you think the following code snippet will give the same result?

```js
from(objectList).pipe(
  map(object => object.employee),
  map(employee => employee.address),
  map(address => address.houseNumber)
)
// vs
from(objectList).pipe(
  pluck('employee', 'address', 'houseNumber')
)

```
Well, the answer is not really the same. Let's take a closer look to see the subtle difference.

## How map operator works

According to the [official documentation](https://rxjs-dev.firebaseapp.com/api/operators/map), here's what map operator does
> Applies a given project function to each value emitted by the source Observable, and emits the resulting values as an Observable.

But that's not the full picture. What will happen if an error occurs in the project function? When I deep dive into the implementation of map operator, here's what I figured out.

```ts
try {
  result = this.project.call(this.thisArg, value, this.count++);
} catch (err) {
  // if error occurs, map will emit an error notification and return
  this.destination.error(err); // <-- this line
  return;
}

```

You can see from the above implementation, if an error occurs in the project function, the map will emit an error notification and your output stream will hang on.

## How pluck operator works

Here's what pluck operator does from the [official docs](https://rxjs-dev.firebaseapp.com/api/operators/pluck)
> Maps each source value (an object) to its specified nested property.

When I read this line, there's a question popping out in my mind, what if the nested property does not exist in the object?

And here's the answer when I consult the source code of pluck operator

```ts
export function pluck<T, R>(...properties: string[]): OperatorFunction<T, R> {
  // if you pass pluck('employee', 'address', 'houseNumber')
  // the length will equal to 3
  const length = properties.length;
  ...
  // under the hood, pluck operator calls map operator,
  // and passes the plucker as projection function
  return (source: Observable<T>) => map(plucker(properties, length))(source as any);
}

```

As you can see, pluck operator calls map operator behind the scene, and passes the plucker as project function. Here’s what plucker does.

```ts
// if you call pluck('employee', 'address', 'houseNumber')
// props will be ['employee', 'address', 'houseNumber']
// and length will be 3
function plucker(props: string[], length: number): (x: string) => any {
  const mapper = (x: string) => {
    let currentProp = x;
    // loop through every passed properties in the list and get the nested value from object
    for (let i = 0; i < length; i++) {
      // if the object doesn't have the specified property, no error will be thrown...
      const p = currentProp != null ? currentProp[props[i]] : undefined; // <--this line
      if (p !== void 0) {
        currentProp = p;
      } else {
        // ...instead, it returns undefined
        return undefined; // <-- this line
      }
    }
    return currentProp;
  };

  return mapper;
}

```

So, from the above code, we can see that the pluck operator will get the nested value from an object based on the property list you provided. For example, you call `pluck('employee', 'address', 'houseNumber')` , it will try to get the value at `object.employee.address.houseNumber`, with the main difference being that it ensures null safety.

*If there's no value inside a nested object, it will return undefined, and your stream will continue to the next emitted value, rather than throwing an error and stopping like a map operator does. This is the main difference between map and pluck operator.*

## Recap

Let me give you a concrete example to recap. Suppose I have the following input data

```ts
const arr = [
  {
    employee: {
      address: {
        houseNumber: 1
      }
    }
  },
  {
    employee: {
      // notice this employee doesn't have address
    }
  },
  {
    employee: {
      address: {
        houseNumber: 3
      }
    }
  },
];

const arr$ = interval(1000).pipe(
  map(index => arr[index]),
  take(3)
);

```

And I have two streams of data

```ts
const streamWithMap = arr$.pipe(
  map(object => object.employee),
  map(employee => employee.address),
  map(address => address.houseNumber)
);

const streamWithPluck = arr$.pipe(
  pluck('employee', 'address', 'houseNumber')
);

```

And here's the visualization of `streamWithMap` and `streamWithPluck` accordingly

![streamWithMap](/img/map-operator.gif)
*streamWithMap*

![streamWithPluck](/img/pluck-operator.gif)
*streamWithPluck*

If I change the `streamWithMap` like this, the result will be the same as when I use pluck

```ts
const streamWithMap = arr$.pipe(
  map(object => object?.employee?.address?.houseNumber),
);

```

## Conclusion

Throughout the article, I have explained in detail what's the main difference between map and pluck operators in RxJS by deep diving into the implementation. I also give an example and marble diagram to illustrate this difference.

I hope you learned something new from the blog. Thanks for reading.
