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

## Conclusion

Throughout the article, I have explained in detail what's the main difference between map and pluck operators in RxJS by deep diving into the implementation. I also give an example and marble diagram to illustrate this difference.

I hope you learned something new from the blog. Thanks for reading.
