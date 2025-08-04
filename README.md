# TypeScript Notes

## TODO
- add notes before Guard Clauses

## Guard Clauses
- quickly narrows types within a function
- classic pattern to deal with undefined & null:
```TypeScript
function processName(name: string | null | undefined) {
  if (name === null || name === undefined) {
    return "";
  }
  // TypeScript knows name is a string here
  return name.toUpperCase();
}
```
Might make sense to throw an error instead:
```TypeScript
function processName(name: string | null | undefined) {
  if (name === null || name === undefined) {
    throw new Error("Name is required");
  }
  // TypeScript knows name is a string here
  return name.toUpperCase();
}
```
if program won't break on empty string, prolly just return that
- i.e., options fields in web apps

## Type Assertion
We can use the `as` keyword to say "trust me, we know it's this type."
```TypeScript
// Property 'toLowerCase' does not exist on type 'string | string[]'
const userId = route.query?.userId.toLowerCase();
```
If we know it will never be an array of strings:
```TypeScript
const userId = (route.query?.userId as string).toLowerCase();
```

Another example:
```TypeScript
type User = {
  id: string;
  name: string;
};

async function getUserRaw(userId: string): Promise<unknown> {
  const response = await fetch(`/api/users/${userId}`);
  return response.json();
}

export async function getUser(userId: string) {
  const data = await getUserRaw(userId);
  // here data is still just "unknown"
  // so we assert it to a User type
  return data as User;
}
```

You can also use this syntax, but `as` feels more clear:
```TypeScript
const userIdRaw = <string>route.query?.userId;
const userId = userIdRaw.toLowerCase();
```

Double assertion: complete insanity:
```TypeScript
const id = 42;

// This works - but is very unsafe!
const userId = id as unknown as string;

// Now TypeScript treats this as a string
console.log(userId.toUpperCase());
// Compiles, but still CRASHES at runtime!
```

## Non-Null Assertion
- (!) - tells the compiler that a value cannot be null or undefined.
- Only use when absolutely certain the value can't be null. Conditional guard clauses are always safer but more verbose.
```TypeScript
interface User {
  id: string;
  name?: {
    first: string;
    last: string;
  };
}

// we don't control the User type (its imported from a library)
// but we know that we always use the `name` property
sentText(user.name!.first);
```

## Random Reducer Example
```TypeScript
export type OrderData = {
  id: string;
  accountType: "free" | "premium";
  amount: number;
  contact: {
    email: string;
    phone: string;
  };
};

function sumOrders(orders: OrderData[]): number {
  return orders.reduce((acc, o) => o.amount + acc, 0);
}
```

## Classes
- #variable: type makes it private in a class
- protected: makes it private to the class but still accessible in subclasses
- abstract ofc makes it not able to be instantiated itself, abstract on a method means it does not have an implementation and must be implemented in subclass

Classes implement interfaces
```TypeScript
interface Vehicle {
  make: string;
  model: string;
}

interface Drivable {
  drive(distance: number): void;
}

class ElectricCar implements Vehicle, Drivable {
  make: string;
  model: string;

  // not required by the interfaces, but it's
  // okay to add extra properties
  private isRunning: boolean = false;

  constructor(make: string, model: string) {
    this.make = make;
    this.model = model;
    this.isRunning = false;
  }

  drive(distance: number): void {
    this.isRunning = true;
    console.log(`Driving ${distance} miles`);
  }
}
```

## Classes vs. Interfaces & Types
Most notable things you can do with classes:
- Have private, protected, static, and abstract members
- Have dedicated constructors
- Have method implementations predefined on all instances
- Support inheritance
On the other hand:
- Type aliases and interface don't have runtime overhead
- Fewer features, simpler
- More flexible, not tied to class implementation
- Interfaces can be extended and merged in ways classes can't

- Essentially, use type aliases when you can, if you need classes then use them.

## Single Source of Truth
This is bad, making a bunch of similar types:
```TypeScript
interface User {
  id: string;
  name: string;
  email: string;
  age: number;
}

interface UserWithoutId {
  name: string;
  email: string;
  age: number;
}
```
Instead, do this (there are other, cleaner strats too below):
```TypeScript
interface UserWithoutId {
  name: string;
  email: string;
  age: number;
}

interface User extends UserWithoutId {
  id: string;
}
```
Try to define types *once* and build type systems that rely on inference and transformation to derive types we need.

## Partial Utility Type
- `Partial<T>` makes all properties of a type optional
```TypeScript
type User = {
  id: string;
  name: string;
  email: string;
};

// Without Partial
function updateUser(
  userId: string,
  userInfo: {
    id?: string;
    name?: string;
    email?: string;
  },
) {
  // ...
}

// With Partial
function updateUser(userId: string, userInfo: Partial<User>) {
  // ...
}
```
But, it only makes the top level properties optional!!!

## Requires Utility Type
- `Required<T>` forces all properties of a type to be required
- Again, only affects top-level properties

## Readonly Utility Type
- `Readonly<T>` all top level properties cannot be reassigned after initialization.

## Record Utility Type
- `Record<K, T>` creates a type with a set of properties K of type T

```TypeScript
// Using string as the key type
type StringKeyDictionary = Record<string, number>;

const karateScores: StringKeyDictionary = {
  "Ralph Macchio": 60,
  "William Zabka": 100,
  "Jackie Chan": 82,
};

// We can add any string key
karateScores["Pat Morita"] = 85;

// But values must be numbers
// Error: Type 'string' is not assignable to type 'number'
karateScores["Eve"] = "A+";
```

It will also give an error if you try to make an object that is missing a key in a union type.
```TypeScript
// Using a union of literal types as keys
type PlayerRole = "tank" | "healer" | "dps";
type RoleCapacity = Record<PlayerRole, number>;

const partyRequirements: RoleCapacity = {
  tank: 1,
  healer: 2,
  dps: 3,
};
```
Here, if we left out `dps: 3,` we would get an error.

## Pick Utility Type
- `Pick<T, K>` creates a new type by selecting a subset of properties from an existing type:

```TypeScript
interface Product {
  id: string;
  name: string;
  price: number;
  description: string;
  category: string;
  inStock: boolean;
  images: string[];
  reviews: { user: string; rating: number; text: string }[];
}

type ProductSummary = Pick<Product, "id" | "name" | "price">;

const productList: ProductSummary[] = [
  { id: "p1", name: "Keyboard", price: 79.99 },
  { id: "p2", name: "Mouse", price: 59.99 },
];

const invalidProduct: ProductSummary = {
  id: "p3",
  name: "Headphones",
  price: 99.99,
  // TSC error:
  // Object literal may only specify known properties, and 'description' does not exist in type 'ProductSummary'.
  description: "Noise cancelling headphones",
};
```

## Omit Utility Type
- Opposite of `Pick<T, K>`
- `Omit<T, K>` excludes a set of properties from an existing type
```TypeScript
interface DatabaseUser {
  id: string;
  username: string;
  email: string;
  passwordHash: string;
  createdAt: Date;
  updatedAt: Date;
}

// Create a safe user representation without sensitive data
type PublicUser = Omit<DatabaseUser, "passwordHash" | "updatedAt">;

function getUserProfile(userId: string): PublicUser {
  // Fetch user from database...
  const dbUser: DatabaseUser = {
    id: userId,
    username: "johndoe",
    email: "john@example.com",
    passwordHash: "$2a$12$...",
    createdAt: new Date("2023-01-15"),
    updatedAt: new Date()
  };

  // Convert to PublicUser (explicit conversion for clarity)
  const publicUser: PublicUser = {
    id: dbUser.id,
    username: dbUser.username,
    email: dbUser.email,
    createdAt: dbUser.createdAt

    // TSC error:
    // Object literal may only specify known properties, and 'passwordHash' does not exist in type 'PublicUser'.
    passwordHash: dbUser.passwordHash,
  };

  return publicUser;
}
```
## Generics
```TypeScript
async function fetchFromApi<T>(url: string): Promise<T | undefined> {
  try {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error("Network response was not ok");
    }
    return await response.json();
  } catch (error) {
    console.error("Error fetching data:", error);
    return undefined;
  }
}

const comments = await fetchFromAPI<Comment[]>(
  "https://api.example.com/posts/1/comments",
);

const user = await fetchFromApi<User>("https://api.example.com/user/1");

const posts = await fetchFromApi<Post[]>("https://api.example.com/posts");
```

Example of a transform function like .map():
```TypeScript
function transform<InputType, OutputType>(
  inputs: InputType[],
  update: (item: InputType) => OutputType,
): OutputType[] {
  const outputs: OutputType[] = [];
  for (const input of inputs) {
    const output = update(input);
    outputs.push(output);
  }
  return outputs;
}

type Human = {
  name: string;
  age: number;
};

const humans: Human[] = [
  { name: "Eren", age: 15 },
  { name: "Mikasa", age: 16 },
  { name: "Armin", age: 15 },
];

const titanTransformer = (human: Human): string => `${human.name} is a titan!`;

const titanNames = transform<Human, string>(humans, titanTransformer);
console.log(titanNames);
// ['Eren is a titan!', 'Mikasa is a titan!', 'Armin is a titan!']
```
## Generic Contraints
```TypeScript
interface HasCost {
  cost: number;
}

function applyDiscount<T extends HasCost>(vals: T[], discount: number): T[] {
  const arr: T[] = [];
  for (const val of vals) {
    val.cost *= discount;
    arr.push(val);
  }
  return arr;
}
```
This applyDiscount function works on any type that has a .code property.

## Type Parameters for Types
Type parameters aren't just limited to functions and methods.
```TypeScript
interface Store<T> {
  get(id: string): T;
  save(id: string, item: T): void;
  list(): T[];
}
// also works with type aliases using
// type Store<T> = { ... }
```
Now store can be anything that implementes the methods.

We can use the store:
```TypeScript
function addAndGetItems<T>(store: Store<T>, id: string, newItem: T): T[] {
  store.save(id, newItem);
  return store.list();
}
```
And, we can create a store dealing with specific types:
```TypeScript
type Product = {
  name: string;
  price: number;
};

const productStore = {
  products: {} as Record<string, Product>,
  get(id: string): Product {
    return this.products[id];
  },
  save(id: string, item: Product): void {
    this.products[id] = item;
  },
  list(): Product[] {
    return Object.values(this.products);
  },
};
```

## Advanced TS - Conditional Types
- Advanced TS features are more useful in library code than application code.

Conditional types let us create new types based on conditions within the type system:
```TypeScript
type NewType = SomeType extends OtherType ? TrueType : FalseType;
```
Simple example:
```TypeScript
type IsString<T> = T extends string ? true : false;

// Usage
type Result1 = IsString<"hello">; // true
type Result2 = IsString<42>;      // false
type Result3 = IsString<string>;  // true
```

TypeScript has built-in conditional types:
```TypeScript
type Extract<T, U> = T extends U ? T : never;
type Exclude<T, U> = T extends U ? never : T;
type NonNullable<T> = T extends null | undefined ? never : T;
```

When is this useful?

Say we have some events:
```TypeScript
type ClickEvent = { type: "click"; x: number; y: number };
type KeyEvent = { type: "key"; key: string };
type MouseMoveEvent = { type: "mousemove"; x: number; y: number };
type FormEvent = { type: "submit"; formId: string };

type Event = ClickEvent | KeyEvent | MouseMoveEvent | FormEvent;
```

Do dynamically create types, we can:
```TS
// Extract is a TS built-in type, this is the implementation:
type Extract<T, U> = T extends U ? T : never;

// This is an example of how it can be used:
type MouseRelatedEvents = Extract<Event, { x: number; y: number }>;
```

Now `MouseRelatedEvents` is equal to:
```TS
type MouseRelatedEvents = ClickEvent | MouseMoveEvent;
```

Using the conditional type, if we add more mouse events, `MouseRelatedEvents` will automatically update rather than us needing it add it to a union type.

## Advanced TS - Infer
```TS
type GetReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
```

```TS
function greet() { return "Hello, world!"; }
function sum(a: number, b: number) { return a + b; }

type GreetReturnType = GetReturnType<typeof greet>; // string
type SumReturnType = GetReturnType<typeof sum>;     // number
```

## Advanced TS - Mapped Types
```TS
type Soldier = {
  name: string;
  age: number;
  branch: "garrison" | "military police" | "survey corps";
};

type OptionalSoldier = {
  [K in keyof Soldier]?: Soldier[K];
};
```
- The keyof operator gets the keys of the Soldier type
- The in keyword iterates over them
- The ? makes each property optional
- The Soldier[K] gets the value type each property maps to
These result in the same type, but all fields are optional.
- The benefit is that when we update Soldier, OptionalSoldier updates too.

You can also use it to change the value type of properties:
```TS
type StringifiedSoldier = {
  [K in keyof Soldier]: string;
};

type StringifiedSoldier = {
  name: string;
  age: string;
  branch: string;
};
```
These are the same^

Mapped Types With Conditionals:

The following filters out any non-string properties.
```TS
type FilteredSoldier = {
  [K in keyof Soldier]: Soldier[K] extends string ? Soldier[K] : never;
};
```

## Running TypeScript locally

tsconfig.json
- configures compiler behavior

Here's a simple example:
```TS
{
  "compilerOptions": {
    "lib": ["esnext"],
    "target": "esnext"
  }
}
```
- `lib` specified the library files to include in compilation. I.E., what APIs are available for us to use.
- `target` specified ECMAScript target version for the JS output.
- `esnext` is great for starting a new project, but probably want to choose a specific version before publishing a project e.g. `es2024`.

Important compiler options:
- `lib:` Add `dom` and `dom.iterable` (note: lowercase) to the list of libraries to allow all the browser APIs if you're writing front-end code.
- `strict:` If `true`, enables all strict type checking options. I strongly recommend it for new projects. You might need to turn it off if you're migrating an existing JS project.
- `skipLibCheck:` If `true`, skips type checking of all declaration files (which means it won't try to type check your infinitely large `node_modules` folder). Drastically speeds up compilation time.
- `verbatimModuleSyntax:` If `true`, simplifies some weirdness with importing and exporting types, basically it forces you to import and export types using the `import type` syntax. I recommend it.
- `esModuleInterop:` If `true`, allows you to use `import` syntax with CommonJS modules. Very useful if you need to work with CommonJS (Node) code.
`moduleDetection:` If set to `force`, will consider everything to be a module, which is what you want in any new project.
`noUncheckedIndexedAccess`: If `true`, adds `undefined` to the type of any indexed access, which can prevent some runtime errors. I recommend it.

Declaration Files `.d.ts`
- They only contain type information - no runtime code allowed.

Here's an example from boot.dev. They use sign-in with Google and Google, per their instructions, wants their JS library in your HTML. Because you still want static type hints in your editor, you do something like this:
```TS
declare global {
  interface Window {
    google: Google;
  }
}

interface Google {
  accounts: {
    id: {
      renderButton: (
        a: HTMLElement,
        b: {
          type?: string;
          theme?: string;
          size?: string;
          text?: string;
          shape?: string;
          width?: number;
        },
      ) => void;
      prompt: () => void;
      cancel: () => void;
      initialize: ({ client_id: string, callback }) => void;
      disableAutoSelect: () => void;
      revoke: (client_id: string, callback) => void;
    };
  };
}

export {};
```
This just says "hey, there's a global variable called `google` one the `window` object, and it has this shape. Now, you can `window.google` in code and get hints.