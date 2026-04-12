---
theme: default
title: JSBT
class: text-center
transition: slide-up 
comark: true
canvasWidth: 800
monacoRunAdditionalDeps:
  - "@cheprasov/jsbt"
layout: quote
---

<img src="./images/cover.jpg" style="width: 100%;" />

---
layout: quote
class: text-center
---

# **JSBT**
## **Binary serialization for real JavaScript object graphs**  
_
## **Alexander Cheprasov**  
### Senior Software Engineer @ Valstro (UK) Ltd  
### Creator of JSBT  

---

## → Context

**A few years ago, at work, we were choosing how to encode data.
We looked at different options:**

- JSON (+ compression)
- Avro
- Protobuf
- Other binary formats

**The requirement was simple:**

→ we needed a binary format for JavaScript data

---

## → So I started thinking:

**Can such a binary format for JavaScript
actually be designed?**

- without schemas
- without codegen
- preserving real JavaScript data (Date, Map, Set, BigInt, ...)
- as simple as JSON.stringify() / JSON.parse()

---

## → Encoding is easy... until it's not

**Encoding data into binary is not that hard**

…pause…

Until you deal with real JavaScript data

Because it’s **not just trees**.

It’s **graphs — with identity, references, and cycles**

---

## → Simple example (tree)  
  
```js {monaco-run}
const userA = { name: "Adam" };
const userB = { name: "Bob" };

userA.friends = [ userB ];

console.log(JSON.parse(JSON.stringify(userA)));
```

✅ Works

---

## → The problem

**Most serialization formats assume data is a tree:**

- nested objects
- no cycles (objects don’t reference each other in loops)
- no shared references (no reuse of the same object)

---

**But JavaScript works differently**

## → JS data is an object graph

- objects can reference each other
- the same object can be reused in multiple places
- cycles can exist (reference loops)

**JSON → tree model**  
**JavaScript → graph model**  

Same data, different model

---

<img src="./images/adam-bob.png" style="width: 100%;" />

---

## → Now it's a graph

```js {monaco-run}
const userA = { name: "Adam" };
const userB = { name: "Bob" };

userA.friends = [ userB ];
userB.friends = [ userA ]; // added reference loop

console.log(JSON.parse(JSON.stringify(userA)));
```

❌ TypeError: **Converting circular structure to JSON**

---

## → Identity problem / shared references
```js
const child = { name: "Matvey" };
const parentA = { name: "Alex",  child };
const parentB = { name: "Irina", child };

const family = { parentA, parentB, child };
console.log(JSON.parse(JSON.stringify(family)));
```
```js
{
  "parentA": {
    "name": "Alex",
    "child": { "name": "Matvey" }
  },
  "parentB": {
    "name": "Irina",
    "child": { "name": "Matvey" }
  },
  "child": { "name": "Matvey" }
}
```
--- 

## → Identity problem / shared references

```js
{
  "parentA": {
    "name": "Alex",
    "child": { "name": "Matvey" }
  },
  "parentB": {
    "name": "Irina",
    "child": { "name": "Matvey" }
  },
  "child": { "name": "Matvey" }
}
```

 ❌ Identity lost

```js
parentA.child !== parentB.child
parentA.child !== child
```

---

## → Key idea

And this is not some edge case. This is normal JavaScript.

JavaScript data is naturally:
- shared references
- circular structures
- complex object graphs

**And this is exactly the problem I wanted to solve.**  

So **JSBT is a binary format that treats JavaScript data as a graph.** Not as a tree.

---

## → JSBT

**JSBT** (JavaScript Binary Transfer) is a binary serialization format designed for JavaScript → JavaScript communication.  

#### **JSBT preserves:**  
- object identity
- shared references
- circular structures
- native JavaScript types (Date, Map, Set, BigInt, TypedArray, ... )
- class instances

---

## → How it works

**Each object gets an identity during encoding**  

- when an object is seen for the first time → it is encoded
- when the same object appears again → it is not encoded again
- instead, a reference to the original object is used

**This allows:**

- shared references (same object reused)
- circular structures (no infinite loops)
- no duplication of data

---

## → Demo

Let’s look at real JavaScript data

---

#### → Demo: How to use JSBT

```js
import { JSBT } from '@cheprasov/jsbt';

const data = {
  id: 123456n,
  createdAt: new Date(),
  set: new Set(['user', 'premium']),
  map: new Map([['answer', 42]]),
  scores: [3.14, 2.71, 1.41, NaN, null, undefined, ,42],
  data: new Uint8Array([1, 2, 3, 4, 5])
};

const encData = JSBT.encode(data);
const decData = JSBT.decode(encData);

console.log(typeof decData.id, decData.id); // bigint, 123456
console.log(typeof decData.set, decData.set); // object, Set (2) {"user", "premium"}
console.log(decData.scores); // [3.14, 2.71, 1.41, NaN, null, undefined, , 42]
console.log(decData.createdAt instanceof Date); // true
```

---

<img src="./images/adam-bob.png" style="width: 100%;" />

---

#### → Demo: Circular structures

```js {monaco-run}
import { JSBT } from '@cheprasov/jsbt';
const userA = { name: "Adam" };
const userB = { name: "Bob" };

userA.friends = [ userB ];
userB.friends = [ userA ]; // added reference loop

const encUserA = JSBT.encode(userA);
console.log(typeof encUserA, encUserA.length); // string, 39

const decUserA = JSBT.decode(encUserA);
console.log(typeof decUserA, decUserA.name, decUserA.friends[0].name); // object, Adam, Bob

const bob = decUserA.friends[0];
console.log(decUserA === bob.friends[0]); // true
```
---

#### → Demo: Identity / shared references

```js {monaco-run}
import { JSBT } from '@cheprasov/jsbt';

const child = { name: "Matvey" };
const parentA = { name: "Alex",  child };
const parentB = { name: "Irina", child };

const family = { parentA, parentB, child };
const decFamily = JSBT.decode(JSBT.encode(family));

console.log(decFamily.parentA.child === decFamily.parentB.child); // true
console.log(decFamily.parentA.child === decFamily.child); // true
```
Same object. Same reference.
---

#### → Demo: Class instances
```ts {monaco-run}
import { JSBT } from '@cheprasov/jsbt';

class User {
    constructor(protected name: string, protected age: number) {
       this.name = name;
       this.age = age;
    }

    getName(): string { return this.name };
    isAdult(): boolean { return this.age >= 18; };
}
const user = new User("Alex", 42);

JSBT.setClassFactories({ User });
const decUser = JSBT.decode(JSBT.encode(user));

console.log(decUser instanceof User, decUser.getName(), decUser.isAdult()); // true, Alex, true
```
---

```js
const baseDate = new Date();
const baseScores = Array.from({ length: 20 }, (_, i) => i * 1.2345);
return Array.from({ length: 10_000 }, (_, i) => ({
    id: i % 100,
    name: `User ${i % 50}`,
    isActive: i % 3 !== 0,
    age: 25 + (i % 5),
    tags: ['node', 'typescript', 'binary', 'serialization']
        .sort(() => Math.random() * 2 - 1).slice(0, 3).sort(),
    scores: baseScores,
    createdAt: baseDate.toISOString(),
    balance: 1000 + (i % 10),
    meta: {
        retries: i % 3,
        source: 'bulk-playground',
        flags: { a: true, b: false, c: true },
        group: `group-${i % 10}`,
    },
}));
```

---

#### → Demo: Structure deduplication (side effect)

JSON - **4.37 MB**  
█ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ 

JSBT - **0.074 MB**  
█  
  
~60× smaller (~2% of JSON)  

**Why**
- no duplication
- reuse of structures

---

## → What JSBT is NOT

**JSBT is not trying to solve every problem**

- not a cross-language protocol  
  (designed for JavaScript ↔ JavaScript)

- not a schema evolution system  
  (no versioning, migrations are application-level)

- not a replacement for Protobuf or Avro  
  (different goals and trade-offs)

- not just a compression tool  
  (size reduction is a side effect of graph encoding)

---

## → FAQ (1/2)

**Why JS-only?**  
→ designed to preserve real JavaScript semantics  
→ identity, references, and native types don’t translate cleanly across languages  

**JSON + gzip?**  
→ compression reduces size  
→ but does not preserve structure (no identity, no cycles)  

---

## → FAQ (2/2)

**Versioning?**  
→ same as JSON  
→ handled at the application level (encode → adapt → use)  

**Safe?**  
→ no eval, no code execution  
→ data-only decoding  
→ zero dependencies (no third-party attack surface)

---

## → When to use

**Use JSBT when:**

- you control both ends (JavaScript ↔ JavaScript)
- you work with complex data structures (Maps, Sets, Dates, BigInts, TypedArrays, ...)
- you need to preserve identity and references (not just values)
- your data contains cycles or shared objects
- your payloads are large and repetitive (logs, analytics, real-time data)

---

## → When NOT to use

**JSBT is probably not the right choice when:**

- you need cross-language communication (use Protobuf / Avro / etc.)
- you rely on schema-first contracts and strict typing
- your data is small and simple (JSON is already enough)
- your main concern is CPU performance on trivial payloads

---

## → Closing

I started with a simple question:

Can we build a binary format for JavaScript  
that is simple, schema-less, and preserves real data?

...

<div v-click><b>JSBT is my answer to that question.</b></div>

---

## → Try it

<div style="display: flex; align-items: center; gap: 40px;">

<div style="flex: 1;">

**NPM:** @cheprasov/jsbt  

**GitHub:** https://github.com/cheprasov/ts-jsbt  
  
**Playground:** https://cheprasov.github.io/ts-jsbt-playground/  

**Linkedin:** https://www.linkedin.com/in/alexandercheprasov/

</div>

<div style="flex: 1; text-align: center;">
  <img src="./images/qr-code.png" width="350" />
  Scan to explore
</div>

</div> 