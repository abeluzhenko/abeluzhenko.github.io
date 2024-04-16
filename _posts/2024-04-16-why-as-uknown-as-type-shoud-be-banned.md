---
layout: post
title: "Why `as unknown as Type` should be banned"
date: 2024-04-16 12:40:00 +0400
categories: typescript
tags: typescript type-casting unknown
---

![as-unknown-as-type](/assets/images/as-unknown-as-type.webp)

There is a common pattern in TypeScript codebases to use `as unknown as Type` for typecasting. I believe that this pattern should be banned. In this article, I will explain why.

<!--more-->

### Unknown type

The unknown type was introduced in [TypeScript 3.0](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-0.html#new-unknown-top-type) and it was expected to be a [more type-safe alternative to `any`](https://github.com/Microsoft/TypeScript/pull/24439)

The crucial difference between the two is that the `unknown` is only assignable to itself, while the any doesn’t have such a limitation.

```typescript
let a: unknown = "unknown";
let b: any = "any";

let c: string;

// Error: Type 'unknown' is not assignable to type 'string'
c = a;

// Fine
c = b;
```

The unknown type seems to be the best choice when we can’t be confident about a variable type:

```typescript
const data: unknown = JSON.parse(unsafeJson);

const result: uknown = eval(externalCode);
```

My rule of thumb here is that any external data has an unknown type by default until it is inferred. E.g.:

```typescript
function isBarable(value: any): value is { bar: () => void } {
  return value && "bar" in value && typeof value.bar === "function";
}

const data: unknown;

if (isBarable(data)) {
  data.bar();
}
```

> See: [Using the unknown type](https://learntypescript.dev/03/l7-unknown)

### Misusing `unknown` type

Sometimes we could see a pattern `as unknown as Type` to be used for typecasting.
This is the most common misusage of the `unknown` type.
In this case `as unknown as Type` works exactly the same way like `as any as Type`:

```typescript
const a: TypeA = { a: "a" };
const b: TypeB = { b: "b" };

// These two are equivalent for the purpose of typecasting
a = b as unknown as TypeA;
a = b as any as TypeA;
```

In the first step, a variable `b` is converted to `unknown` and then, to `any`.
We already know that the only difference between the two is that `unknown` can't be assigned to any known type without type inference of casting. And the casting here is crucial. Since both `any` and `unknown` can be type-casted to anything, this last part of the expression just annihilates this difference.
This approach could even suggested by a TSServer as a "solution" for non-overlapping types casting:

![TSServer suggestion](/assets/images/as-unknown-as-type-tsserver-suggestion.webp)

It seems like a hacky way to use `any` without `any`, when the `es-lint` `no-explicit-any` rule is set.

![As uknowns Scooby meme](/assets/images/as-unknown-as-type-meme.webp)

> If you're confused about these "non-overlapping types conversion" errors, that's completely fine, they're confusing.
> Here is an article that should help: [Understanding TS type casting](/typescript/2024/02/24/understanding-ts-typecasting.html)

But while the usage of `any` requires us to explicitly disable the rule, `as unknown as Type` doesn't.
Hence, it isn't so easily trackable. Also, using both approaches in a single codebase makes the code less consistent:

```typescript
// rules disables are easy to track
// eslint-disable-next-line @typescript-eslint/no-explicit-any
a = b as any as TypeA;

// a bit harder to track
a = b as unknown as TypeA;
```

### Typecasting problem

One may say, that the problem here isn't an approach for types casting, but the casting itself.
One of the goals of a type system is to prevent errors in the earliest stages. Using type casts we create blind spots, unreachable for checking and just deprive a type system from doing its job properly.
This way we may end up with a useless type system along with an obligation to write types.
When a type system seems useless for developers, it doesn't incentivize them to write quality typings, and the vicious circle is closing.

But sometimes, casting is necessary. Let's take, for example, a unit-test code:

```typescript
type Bar = {
  name: string;
  bar: () => void;
};

const mock = {
  bar: () => {},
} as Bar;

const result = foo(mock);

expect(result).toBeTruthy();
```

We use typecasting to pass a simple mock to the tested function. The mock contains only the fields that are necessary for the test case and we don't need its type to be strictly equal to the one, expected by the function. So, the typecasting seems rational so far.

But then we need to check if some method of the passed object is called, so we change the mock by assigning a `jest.fn()` to the appropriate field:

```typescript
const mock = {
  bar: jest.fn(),
} as Bar;

foo(mock);

expect(mock.bar).toHaveBeenCalled();
```

This change leads to the following error:

```
Conversion of type '{ bar: jest.Mock<any, any, any>; }' to type 'Bar' may be a mistake because neither type sufficiently overlaps with the other. If this was intentional, convert the expression to 'unknown' first.
Property 'name' is missing in type '{ bar: jest.Mock<any, any, any>; }' but required in type 'Bar'.ts(2352)
```

Of course, we could add a `name` property to the mock, even if we don't expect it to be read in this particular case, but in most cases it isn't that simple:

```typescript
const el = { testId: "testId" } as HTMLElement;
```

```
Conversion of type '{ testId: string; }' to type 'HTMLElement' may be a mistake because neither type sufficiently overlaps with the other. If this was intentional, convert the expression to 'unknown' first.
Type '{ testId: string; }' is missing the following properties from type 'HTMLElement': accessKey, accessKeyLabel, autocapitalize, dir, and 290 more.
```

There is nothing easier than just to follow the suggestion and cast the mock to `unknown` first, which may seem like an appropriate solution right now but creates a breach in the type system's defense.

Imagine, someone would decide to refactor the type and change its signature renaming one of the argument fields.

```typescript
type Bar = {
  name: string;

  // old field
  // bar: () => void

  // new field
  fooBar: () => void;
};

// no type errors
const mock = {
  bar: jest.fn(),
} as unknown as Bar;
```

Hopefully, the test is going to fail, but the type check still passes. This way we just have disabled the first line of defense. Since unit tests are always slower than a type check, the feedback time for a developer is increased.
This is one of the plenty of tiny pitfalls that could spoil DX and increase TTM.

### Summary

- `unknown` is a legal way to put some data into an "untyped leprosarium". There is the only way to get out of there: type inference;
- using `an any as Type` instead of `as unknown as Type` is a more explicit way to say that there is a hacky type-casting;
- both options make the code less maintainable. It is always worth checking why types do not overlap when they are supposed to;
