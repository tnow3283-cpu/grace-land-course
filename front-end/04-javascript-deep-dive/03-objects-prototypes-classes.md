# Objects, Prototypes & Classes

JavaScript's object system is **prototype-based**. Classes are syntax over prototypes, not a separate inheritance model. Understanding the prototype chain explains method lookup, `instanceof`, and most "weird" object behavior.

> **How to run this material**
> Every snippet below is self-contained and verified with Node.js 20. Save a snippet to a file and run `node file.js`, or paste it into the browser DevTools console. Snippets use `const`/`class`, so run them one at a time rather than pasting the whole page into a single script.

---

## 1. Creating Objects

Four ways to make an object. All four end up as plain objects whose method lookup walks a prototype chain.

```js
// 1) Object literal
const user = {
  name: "Ada",
  age: 36,
  greet() { return `Hi, ${this.name}`; },
};
console.log(user.greet()); // "Hi, Ada"

// 2) Constructor function (legacy pattern)
function LegacyUser(name, age) {
  this.name = name;
  this.age = age;
}
LegacyUser.prototype.greet = function () {
  return `Hi, ${this.name}`;
};
console.log(new LegacyUser("Grace", 30).greet()); // "Hi, Grace"

// 3) Object.create — inherit from an existing object
const admin = Object.create(user);
admin.role = "admin";
console.log(admin.name, admin.role);            // "Ada admin" (name is inherited)
console.log(Object.hasOwn(admin, "name"));      // false
console.log(Object.hasOwn(admin, "role"));      // true

// 4) Class syntax (modern)
class Member {
  constructor(name) { this.name = name; }
  greet() { return `Hi, ${this.name}`; }
}
console.log(new Member("Linus").greet()); // "Hi, Linus"
```

`Object.create(user)` makes `admin` inherit directly from the `user` **object**, not from `user`'s constructor. That is the clearest way to see what a prototype really is.

---

## 2. Property Access and Descriptors

Dot and bracket access are the same lookup. Missing properties return `undefined` instead of throwing; optional chaining stops the walk safely.

```js
const config = { theme: "dark", nested: null };

// Dot vs bracket access
console.log(config.theme);         // "dark"
console.log(config["theme"]);      // "dark"
console.log(config.missing);       // undefined

// Optional chaining — safe access
console.log(config.nested?.value); // undefined

// Two ways to check existence
console.log("theme" in config);                 // true (own or inherited)
console.log(Object.hasOwn(config, "theme"));    // true (own only)
console.log(Object.hasOwn(config, "toString")); // false (inherited)

// A locked-down property
const obj = {};
Object.defineProperty(obj, "id", {
  value: 42,
  writable: false,
  enumerable: false,
  configurable: false,
});

console.log(obj.id); // 42

// Strict mode turns the failed write into a TypeError
try {
  (() => { "use strict"; obj.id = 99; })();
} catch (e) {
  console.log(e.name); // "TypeError"
}
console.log(obj.id);           // 42
console.log(Object.keys(obj)); // [] — non-enumerable

console.log(Object.getOwnPropertyDescriptor(obj, "id"));
// { value: 42, writable: false, enumerable: false, configurable: false }

// Gotcha: defineProperty defaults ALL flags to false
const defaults = {};
Object.defineProperty(defaults, "x", { value: 1 });
console.log(Object.getOwnPropertyDescriptor(defaults, "x"));
// { value: 1, writable: false, enumerable: false, configurable: false }
```

| Flag | Default in object literals | Default in `defineProperty` | Meaning |
| :--- | :--- | :--- | :--- |
| `writable` | `true` | `false` | Can be reassigned |
| `enumerable` | `true` | `false` | Appears in `for...in` / `Object.keys` |
| `configurable` | `true` | `false` | Can be deleted or redefined |

The strict-mode demo is wrapped in an IIFE because plain Node/browser scripts are sloppy mode by default. ES modules and class bodies are always strict.

---

## 3. The Prototype Chain

Every object has an internal `[[Prototype]]` link. Property lookup walks this chain until it finds the property or reaches `null`.

```
rabbit ──[[Prototype]]──▶ animal ──[[Prototype]]──▶ Object.prototype ──▶ null
```

```js
const animal = {
  eats: true,
  walk() { return "walking"; },
};

const rabbit = Object.create(animal);
rabbit.jumps = true;

// Own vs inherited
console.log(rabbit.jumps);  // true (own)
console.log(rabbit.eats);   // true (inherited)
console.log(rabbit.walk()); // "walking" (inherited)

console.log(Object.getPrototypeOf(rabbit) === animal);            // true
console.log(Object.getPrototypeOf(animal) === Object.prototype);  // true
console.log(Object.getPrototypeOf(Object.prototype));             // null — end of chain

// Own-property iteration vs inherited-inclusive iteration
console.log(Object.keys(rabbit));           // ["jumps"]
console.log(Object.hasOwn(rabbit, "eats")); // false

const seen = [];
for (const key in rabbit) seen.push(key);
console.log(seen); // ["jumps", "eats", "walk"] — inherited ones included

// Guard when you only want own keys
const own = [];
for (const key in rabbit) {
  if (Object.hasOwn(rabbit, key)) own.push(key);
}
console.log(own); // ["jumps"]
```

> **Gotcha:** `for...in` includes inherited enumerable properties. Use `Object.keys` or add an `Object.hasOwn` guard when you mean "own data".

---

## 4. Classes

A class is a cleaner way to write a constructor plus its prototype methods. Here is inheritance done correctly: the base class owns `#sound`, and the subclass passes its value up through `super`.

```js
class Animal {
  #sound; // private field — lives on the instance, not on the prototype

  constructor(name, sound = "...") {
    this.name = name;
    this.#sound = sound;
  }

  speak() {
    return `${this.name} says ${this.#sound}`;
  }

  get description() {
    return `${this.constructor.name}: ${this.name}`;
  }

  static create(...args) {
    return new this(...args);
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name, "Woof"); // must run before touching `this`
    this.breed = breed;
  }

  speak() {
    return `${super.speak()} (${this.breed})`;
  }
}

const rex = new Dog("Rex", "Labrador");
console.log(rex.speak());            // "Rex says Woof (Labrador)"
console.log(rex.description);        // "Dog: Rex"
console.log(rex instanceof Dog);     // true
console.log(rex instanceof Animal);  // true

// `static create` uses `new this` so it builds the right subclass
console.log(Dog.create("Fido", "Poodle").speak()); // "Fido says Woof (Poodle)"

// Private fields never show up as normal properties
console.log(Object.keys(rex)); // ["name", "breed"]
console.log(rex["#sound"]);    // undefined

// A method that touches a private field refuses a foreign object
try {
  Animal.prototype.speak.call({ name: "Fake" });
} catch (e) {
  console.log(e.name); // "TypeError"
}
```

Two things that surprise people:

- **Private names are lexical, not string properties.** `#sound` in `Animal` and `#sound` in `Dog` would be two *different* private slots even though the spelling matches. That is why the field lives in one class here and is passed through `super`, instead of being redeclared in both.
- **`super.speak()` calls the parent method with `this` still set to the `Dog` instance.** The parent reads its own private slot, which the subclass filled via `super(name, "Woof")`.

### Class fields, static members, and static blocks

```js
class Counter {
  count = 0;            // public instance field
  static instances = 0; // public static field
  #secret = "hidden";   // private field

  constructor() {
    Counter.instances++;
  }

  increment() {
    this.count++;
    return this; // enables chaining
  }

  reveal() {
    return this.#secret;
  }

  static {
    // Runs once when the class is defined
    console.log("Counter class initialized");
  }
}

const c = new Counter().increment().increment();
console.log(c.count);            // 2
console.log(c.reveal());         // "hidden"
console.log(Counter.instances);  // 1

// Only public instance fields are enumerable own properties
console.log(Object.keys(c));       // ["count"]
console.log(Object.keys(Counter)); // ["instances"]

// Methods live on the prototype, not on each instance
console.log(typeof c.increment);                            // "function"
console.log(Object.hasOwn(c, "increment"));                 // false
console.log(Object.hasOwn(Counter.prototype, "increment")); // true

// Prototype methods are non-enumerable (unlike object-literal methods)
console.log(Object.keys(Counter.prototype)); // []
console.log(
  Object.getOwnPropertyDescriptor(Counter.prototype, "increment").enumerable
); // false
```

> **Important:** Class bodies run in strict mode. Class declarations are **not hoisted** like function declarations — they sit in the temporal dead zone (TDZ) until evaluation reaches them.

---

## 5. Inheritance vs Composition

Inheritance models **is-a**. Composition models **has-a / can-do** and avoids fragile, deep trees.

```js
// IS-A: inheritance
class User {
  constructor(name) { this.name = name; }
}
class Admin extends User {
  constructor(name) {
    super(name);
    this.role = "admin";
  }
}

const admin = new Admin("Ada");
console.log(admin instanceof User, admin.role); // true "admin"

// HAS-A / CAN-DO: composition
const canLog = (state) => ({
  log() { console.log(state.name); },
});

const canSerialize = (state) => ({
  toJSON() { return { ...state }; },
});

const createUser = (name) => {
  const state = { name };
  return { ...state, ...canLog(state), ...canSerialize(state) };
};

const guest = createUser("Grace");
guest.log();                         // "Grace"
console.log(JSON.stringify(guest));  // {"name":"Grace"} — toJSON hides the functions
console.log(guest instanceof User);  // false — no inheritance involved
```

> **Rule of thumb:** Use inheritance for genuine taxonomic relationships. Use composition for capabilities. Deep inheritance trees become fragile because a change near the top ripples everywhere.

---

## 6. `new`, `instanceof`, and `constructor`

```js
function Person(name) {
  this.name = name;
}
Person.prototype.greet = function () {
  return this.name;
};

const ada = new Person("Ada");

console.log(ada.greet());                          // "Ada"
console.log(ada.constructor === Person);           // true (inherited)
console.log(ada instanceof Person);                // true
console.log(Person.prototype.isPrototypeOf(ada));  // true

// Doing by hand exactly what `new` does
const manual = Object.create(Person.prototype); // 1 + 2: create and link
Person.call(manual, "Grace");                   // 3: run constructor with `this`
console.log(manual.greet());                    // "Grace" (4: return the object)

// A constructor that returns an object overrides the fresh `this`
function Weird() {
  this.ignored = true;
  return { replaced: true };
}
console.log(new Weird()); // { replaced: true }
```

What `new` does, step by step:

1. Creates a new empty object.
2. Links its `[[Prototype]]` to `Person.prototype`.
3. Calls `Person` with `this` bound to the new object.
4. Returns that object — **unless** the constructor explicitly returns a different object.

`instanceof` is just a prototype-chain walk: it asks "is `Person.prototype` anywhere on this object's chain?".

> **Gotcha:** Because `instanceof` checks prototypes, not names, it fails across different realms (iframes) and can be customized via `Symbol.hasInstance`.

---

## 7. Copying Objects

A spread or `Object.assign` copy is **shallow**: nested objects are shared by reference.

```js
const original = { name: "Ada", address: { city: "London" } };

const shallow1 = { ...original };
const shallow2 = Object.assign({}, original);

// Nested objects are shared, not duplicated
console.log(shallow1.address === original.address); // true
shallow1.address.city = "Paris";
console.log(original.address.city);                 // "Paris"

// Deep copy with structuredClone
const deep = structuredClone(original);
deep.address.city = "Tokyo";
console.log(original.address.city); // "Paris" — untouched
console.log(deep.address.city);     // "Tokyo"

// Types the JSON trick would destroy
const rich = {
  when: new Date("2020-01-01"),
  tags: new Set(["a", "b"]),
  meta: new Map([["k", 1]]),
};
const copy = structuredClone(rich);
console.log(copy.when instanceof Date); // true
console.log(copy.tags instanceof Set);  // true
console.log(copy.meta instanceof Map);  // true

// structuredClone throws on things it cannot copy
try {
  structuredClone({ fn() {} });
} catch (e) {
  console.log(e.name); // "DataCloneError"
}
```

| Approach | Depth | Keeps `Date`/`Map`/`Set` | Handles cycles | Functions |
| :--- | :--- | :--- | :--- | :--- |
| `{ ...obj }` / `Object.assign` | shallow | keeps the same references | n/a | shares the reference |
| `structuredClone` | deep | yes | yes | **throws** `DataCloneError` |
| `JSON.parse(JSON.stringify())` | deep | no (becomes strings/`{}`) | no (throws) | dropped silently |

> **Prefer `structuredClone`** for deep copies. It also cannot clone DOM nodes. If a value contains a function, it does not skip it — it throws.

---

## 8. Immutability

`Object.freeze` is shallow. It locks the object's own properties, but nested objects and arrays remain mutable.

```js
// Freeze is shallow
const frozen = Object.freeze({ name: "Ada", tags: ["a"] });

// Strict mode turns a failed write into a TypeError
try {
  (() => { "use strict"; frozen.name = "Grace"; })();
} catch (e) {
  console.log(e.name); // "TypeError"
}
console.log(frozen.name); // "Ada"

frozen.tags.push("b"); // allowed — the nested array is not frozen
console.log(frozen.tags); // ["a", "b"]

// Immutable updates: create new objects/arrays instead of mutating
const user = { name: "Ada", age: 36 };
const older = { ...user, age: 37 };
console.log(older);    // { name: "Ada", age: 37 }
console.log(user.age); // 36

const list = [1, 2, 3];
console.log([...list, 4]);                        // [1, 2, 3, 4]
console.log(list.filter((n) => n !== 2));         // [1, 3]
console.log(list.map((n) => (n === 2 ? 20 : n))); // [1, 20, 3]
console.log(list);                                // [1, 2, 3] unchanged
```

> **Why it matters:** Immutable updates make state changes predictable and enable cheap reference equality checks (`prev === next`) in UI libraries and memoized components.

---

## 9. Object Utilities

```js
const obj = { a: 1, b: 2 };

console.log(Object.keys(obj));    // ["a", "b"]
console.log(Object.values(obj));  // [1, 2]
console.log(Object.entries(obj)); // [["a", 1], ["b", 2]]

const pairs = [["x", 1], ["y", 2]];
console.log(Object.fromEntries(pairs)); // { x: 1, y: 2 }

// One-level "deep" merge without a library
const defaults = { theme: "light", nested: { fontSize: 12, bold: false } };
const options = { nested: { bold: true } };
const merged = {
  ...defaults,
  ...options,
  nested: { ...defaults.nested, ...options.nested },
};
console.log(merged);
// { theme: "light", nested: { fontSize: 12, bold: true } }

// Map <-> Object
const map = new Map(Object.entries({ a: 1, b: 2 }));
console.log(map.get("b"));            // 2
console.log(Object.fromEntries(map)); // { a: 1, b: 2 }
```

Spreading only goes one level. Notice that `nested` had to be merged explicitly — otherwise `options.nested` would have completely replaced `defaults.nested`.

---

## 10. JSON

`JSON.stringify` is lossy and `JSON.parse` is safe. Know exactly what gets dropped.

```js
const json = JSON.stringify({ name: "Ada", age: 36 }, null, 2);
console.log(json);
/*
{
  "name": "Ada",
  "age": 36
}
*/
console.log(JSON.parse(json)); // { name: "Ada", age: 36 }

// What JSON silently drops or rewrites
const tricky = {
  fn: () => {},
  missing: undefined,
  sym: Symbol("s"),
  when: new Date("2020-01-01"),
  nan: NaN,
  inf: Infinity,
};
console.log(JSON.stringify(tricky));
// {"when":"2020-01-01T00:00:00.000Z","nan":null,"inf":null}

// Circular references throw
const loop = {};
loop.self = loop;
try {
  JSON.stringify(loop);
} catch (e) {
  console.log(e.name); // "TypeError"
}

// Custom serialization with toJSON
const account = {
  name: "Ada",
  password: "secret",
  toJSON() {
    return { name: this.name }; // password omitted
  },
};
console.log(JSON.stringify(account)); // {"name":"Ada"}
```

- Functions, `undefined`, and symbols are omitted.
- Dates become ISO strings (via `Date.prototype.toJSON`).
- `NaN` and `Infinity` become `null`.
- Circular references throw a `TypeError`.
- If an object has `toJSON()`, JSON uses its return value instead.

> **Security:** Never feed `JSON` into `eval` or `new Function`. `JSON.parse` only produces data; executable evaluation produces vulnerabilities.

---

## ✅ Checkpoint

- [ ] What is the difference between an object's own property and an inherited one?
- [ ] How does the prototype chain resolve a method call?
- [ ] What does `new` actually do?
- [ ] Why is `Object.freeze` shallow?
- [ ] When should you prefer composition over inheritance?
- [ ] What does `structuredClone` handle that `JSON.parse(JSON.stringify())` does not?

<details>
<summary>Answers</summary>

1. **Own vs inherited:** an own property is stored directly on the object; an inherited one is found by walking the object's `[[Prototype]]` chain. `Object.hasOwn` returns `true` only for own properties.
2. **Method lookup:** the engine checks the object, then its prototype, then that prototype's prototype, and so on until it finds the property or reaches `null`. Calling a method found this way still uses the original receiver as `this`.
3. **`new`:** create a fresh object, link it to `Fn.prototype`, run `Fn` with `this` set to the new object, then return that object (unless the constructor returns a different object).
4. **Freeze is shallow:** `Object.freeze` makes the object's own properties non-writable and non-configurable, but it does not recurse into nested objects or arrays.
5. **Composition over inheritance:** when the relationship is a capability ("can log", "can serialize") rather than a taxonomy ("a Dog is an Animal"), or when sharing behavior across unrelated types.
6. **`structuredClone` vs JSON:** it preserves `Date`, `Map`, `Set`, typed arrays, `ArrayBuffer`, and handles circular references. JSON converts dates to strings, turns Maps/Sets into `{}`, drops functions/`undefined`, and throws on cycles.

</details>
