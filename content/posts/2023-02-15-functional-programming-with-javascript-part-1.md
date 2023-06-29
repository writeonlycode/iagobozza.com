---
title: "Functional Programming with JavaScript: Iterating over Arrays"
author: "Iago Bozza"
categories: ["JavaScript"]
date: "2023-02-15"
slug: "functional-programming-with-javascript-part-1"
summary: >
  Functional programming is a programming paradigm that encourages recursive
  functions and structures, immutability, and the use of functions as values.
  What are the differences between these approaches? Which one is better? What
  are the advantages and disadvantages of each? Which one should we use?
draft: true
---

# What is Functional Programming?

Let's start with a rough characterization of functional programming: functional
programming is a programming paradigm that encourages recursive functions and
structures, immutability, and the use of functions as values.

In a functional programming course (like ...), it's usual to see recursive
algorithms to traverse a list or a tree. Take a typical linked list like this
one:

```javascript
function LinkedList(value, next) {
  this.value = value;
  this.next = next;
}
```

In a non-functional programming context, if we want to iterate over each
element of the list and print its value, we would use a `while` loop to code
something like this:

```javascript
function printEachNodeLoop(node) {
  let currentNode = node;

  while (node !== null) {
    console.log(node.value);
    currentNode = node.next;
  }
}
```

The code above is paradigmatically non-functional: we use a mutable reference
to the current node and a loop. In a functional programming context, to achieve
the same result, we would drop the use of mutable references and use a
recursive function like this:

```javascript
function printEachNodeRecursive(node) {
  if (node === null) {
    return;
  } else {
    console.log(node.value);
    printEachNodeRecursive(node.next);
  }
}
```

What is the difference? Which approach is better? What are the advantages and
disadvantages of each approach? Which one should we use?


# Iterating Over Linked Lists: Time

Let's start with a few tests: how much time does it take to perform the same
iteration over a linked list using the recursive approach and using the loop
approach?

Let's define a linked list structure, and create a bunch of instances of it, so
we can measure how much time it takes to go over it:

```javascript
function LinkedList(value, next) {
  this.value = value;
  this.next = next;
}

const linkedListA = new LinkedList(0, null);
let currentLinkedList = linkedListA;

for (let i = 1; i < 8801; i++) {
  currentLinkedList.next = new LinkedList(i, null);
  currentLinkedList = currentLinkedList.next;
}
```

Now, let's print the value of each node using the loop approach, and let's keep
track of the time it takes to do it:

```javascript
function forEachLoop(ll) {
  let current = ll;

  while (current !== null) {
    console.log(current.value);
    current = current.next;
  }
}

console.time("forEachLoop");
forEachLoop(linkedListA)
console.timeEnd("forEachLoop");
```

Executing the code above 3 times yielded the results: **98.896ms**,
**96.181ms**, and **106.877ms**. What about the recursive approach? Let's do
the same thing and see how the results stack up. The code will be the
following:

```javascript
function forEachRecursion(ll) {
  if (ll === null) {
    return;
  } else {
    console.log(ll.value);
    forEachRecursion(ll.next);
  }
}

console.time("forEachRecursion");
forEachRecursion(linkedListA);
console.timeEnd("forEachRecursion");
```

Executing the code above 3 times yielded the results: **106.865ms**,
**106.125ms**, and **110.733ms**. Not bad, right? It seems that the loop
approach and the recursive approach work equally well in this case.


# Iterating Over Linked Lists: Stack Size

There is one problem, though: if we increase the size of the linked list from
8818 to 18818, the loop approach takes longer, but the recursive approach
breaks with this exception:

```bash
node:internal/util/inspect:765
function formatValue(ctx, value, recurseTimes, typedArray) {
                    ^

RangeError: Maximum call stack size exceeded
    at formatValue (node:internal/util/inspect:765:21)
    at inspect (node:internal/util/inspect:364:10)
    at formatWithOptionsInternal (node:internal/util/inspect:2291:40)
    at formatWithOptions (node:internal/util/inspect:2153:10)
    at console.value (node:internal/console/constructor:339:14)
    at console.log (node:internal/console/constructor:376:61)
    ...
```

The loop approach simply goes through each one of the nodes of the linked list
and print its value. When the loop moves to the next value, it just discards
the reference to the last node and replaces it with a reference to the new
node.

The recursive approach, however, calls one function for each node, and the
first call doesn't return until we reach the last node. This means that if we
have a linked list with 8818 nodes, we will have 8818 pending function calls
waiting to return. But if we have 18818 nodes, we will exceed the limit of
memory available for pending function calls before we reach the last node.

The loop approach clearly seems superior in this case.


# Iterating Over Arrays

In JavaScript, however, since we have dynamic arrays, we don't really need
linked lists. The same difference applies though: we can use either loops and
mutability or recursion and immutability.

```javascript
function forEachLoop(a) {
  for (let i = 0; i < a.length; i++) {
    console.log(a[i]);
  }
}
```

```javascript
function forEachRecursion(a) {
  if (a.length == 0) {
    return;
  } else {
    console.log(a[0]);
    forEachRecursion(a.slice(1));
  }
}
```


```javascript
let a = new Array(8018);

function forEachLoop(a) {
  for (let i = 0; i < a.length; i++) {
    console.log(a[i]);
  }
}

console.time("forEachLoop");
forEachLoop(a);
console.timeEnd("forEachLoop");
```

Results: 

With recursion.

```javascript
let a = new Array(8018);

function forEachRecursion(a) {
  if (a.length == 0) {
    return;
  } else {
    console.log(a[0]);
    forEachRecursion(a.slice(1));
  }
}

console.time("forEachRecursion");
forEachRecursion(a);
console.timeEnd("forEachRecursion");
```


# Stack Overflow


```javascript
let a = new Array(10180);

function forEachLoop(a) {
  for (let i = 0; i < a.length; i++) {
    console.log(a[i]);
  }
}

console.time("forEachLoop");
forEachLoop(a);
console.timeEnd("forEachLoop");
```

```javascript
let a = new Array(10180);

function forEachRecursion(a) {
  if (a.length == 0) {
    return;
  } else {
    console.log(a[0]);
    forEachRecursion(a.slice(1));
  }
}

console.time("forEachRecursion");
forEachRecursion(a);
console.timeEnd("forEachRecursion");
```

# comments

These results are to be expected: when doing things recursively, for each
iteration, we add a new frame to the stack, and this takes time and memory.
Since node has a limit to the size of the stack, when we cross this limit, we
get the error above.

What are the advantages of doing things recursively and with immutability,
then?

In functional languages, the compiler typically optimizes for tail recursion:
when a recursive call appears in tail position, the compiler typically sets
things up in a way that the previous frame in the stack is reused for the new call
This optimizes both performance and memory usage.

JS, however, doesn't have, and will never have, tail recursion optimizations.
So, the recursive approach is doomed to be inferior in terms of performance and
memory usage.
