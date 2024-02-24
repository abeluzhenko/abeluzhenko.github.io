> Disclaimer: typecasts lead to losing type information, I'd strongly recommend avoiding them, whenever possible.

There are cases when typecasting is necessary or even preferable. For example, that could be casting mocks in tests. Let's take a function that accepts an HTML element and sets up some attributes:

```typescript
function setElementAttributes(element: HTMLElement): void
```

Assuming, we want to cover it with some unit tests. And, for some reason, we need to validate its behavior to be sure that appropriate methods were called with expected arguments. It seems rational to mock an argument of the function in a test case, instead of passing a real HTML element:

```typescript
const element = {setAttributes: jest.fn()} as HTMLElement

setElementAttributes(element)

expect(element.setAttributes).toHaveBeenCalled()
```

The mock contains just enough for this test case, but the code above generates the following TypeScript error:

```
Conversion of type '{ setAttribute: jest.Mock<any, any, any>; }' to type 'HTMLElement' may be a mistake because neither type sufficiently overlaps with the other. If this was intentional, convert the expression to 'unknown' first.  
Type '{ setAttribute: Mock<any, any, any>; }' is missing the following properties from type 'HTMLElement': accessKey, accessKeyLabel, autocapitalize, dir, and 289 more.
```

> The typecasting mechanism has sitting belts: TS doesn't allow typecasting for non-overlapping types.

It may be very appealing just to follow the suggestion above and add `as unknown` before casting the mock to the expected type:

```typescript
const element = {setAttributes: jest.fn()} as unknown as HTMLElement
```

But this will disable any type checks for the object and it will be much harder to reflect future changes of the interface inside the test.

So, where does this TS error come from, and how do we deal with it? Isn't that we, who are the masters of our code? How dare this TypeScript to object us?

To understand what is happening, let's decompose how type-casting works in TypeScript.

Here are types `TypeA` and `TypeB`:

```typescript
type TypeA {
    a: 'a'
}

type TypeB {
    b: 'b'
}
```

Let's try to cast them to each other:

```typescript
const a: TypeA = {a: 'a'}
const b: TypeB = {b: 'b'}

const bToA = b as TypeA // Error: types don't overlap
```

We're getting an error, saying the casting is invalid, since the types don't overlap.

On the other hand, the following casting is valid:

```typescript
const empty = {} as TypeA // This is fine
```

But when we add some field to an empty object, there is a type error again:

```typescript
const notEmpty = {extraField: true} as TypeA // Error: types don't overlap
```

That may seem a bit weird. Why an empty object can be type-casted into `TypeA`, but an object with some extra field can't?

When we cast `TypeA to TypeB`, TS checks whether `TypeA` can be converted to `TypeB` **or** if `TypeB` can be converted to `TypeA`.

> The conversion is impossible for types that **are not overlapping**. It means that **no value can have both types simultaneously**.

In a case when both conversions are impossible, TS shows an error, that is related to the **first** conversion attempt, which could be really misleading:

```
Conversion of type '{ extraField: boolean; }' to type 'TypeA' may be a mistake because neither type sufficiently overlaps with the other. If this was intentional, convert the expression to 'unknown' first.

Property 'a' is missing in type '{ extraField: boolean; }' but required in type 'TypeA'
```

> There is a [suggestion to change this error message](https://github.com/microsoft/TypeScript/issues/47361), with more details.

Let's return to our example again:

```typescript
const empty = {} as TypeA
```

Firstly, TS tries to convert an empty object to `TypeA`. These types are not compatible since the empty object missing a required field `a` from `TypeA`. Next, it tries to convert `TypeA` to `{}`. This is a valid conversion since `TypeA` could be considered as an empty object with some extra fields.

Now let's add an extra field to our object:

```typescript
const notEmpty = {extraField: true} as TypeA
```

Again, TS tries to convert the object to `TypeA`. Since it is missing field `a` from `TypeA`, this conversion won't work. The next step is the conversion of `TypeA` to `{extraField: true}` type. Since `TypeA` misses `extraField`, this conversion is also impossible, and TS says about it, but refers to the first conversion attempt:

```
Property 'a' is missing in type '{ extraField: boolean; }' but required in type 'TypeA'
```

Keeping that in mind helps to understand TS complaints and fix them without unsafe type-casts:

```typescript
const el = {setAttributes: () => {}} as HTMLElement // Error
```

The type of `setAttributes` is not assignable to an appropriate type of `HTMLElement`, since it lacks function parameters. An empty function can't be type-casted into a parametrized one:

```typescript
const el = {setAttributes: (name, value) => {}} as HTMLElement // Fine
```

Just adding the parameters enables the casing and fixes the error. We don't even need to add the parameter types since they will be inferred by TS.

But what about Jest mocks? We have started with an example of casting a mock object:

```typescript
const element = {setAttributes: jest.fn()} as HTMLElement

setElementAttributes(element)

expect(element.setAttributes).toHaveBeenCalled()
```

Since jest's `Mock` is not assignable to `(name: string, value: string) => void`, the type that `HTMLElement['setAttribute']` has, there is a type error. What can we do about it?

The first option is to use `jest.spyOn` to set up a mock:

```typescript
const element = {setAttributes: (name, value) => {}} as HTMLElement

setElementAttributes(element)

const spy = jest.spyOn(element, 'setAttributes')
```

The second is to narrow the method's type since `Mock` is convertible to any function:

```typescript
const element = {
    setAttributes: jest.fn() as HTMLElement['setAttributes'] 
} as HTMLElement
```

Both options won't require any unsafe typecasts and we aren't going to lose any type data.

Of course, the examples here are simplified, and real-life cases could be much more tricky. Here's a little hack that could make resolving typecasting issues a bit easier.

Because TS will always show an error hint related to the first conversion attempt, we can force it to show the more informative one, just by reverting an expression:

```typescript
type FooBar = {
    foo: 'foo',
    bar: 'bar'
    fooBar: (value: string) => number
}

const a = { fooBar:() => 0 } as FooBar
```

The error hint doesn't make much sense:

```
Conversion of type '{ fooBar: () => number; }' to type 'FooBar' may be a mistake because neither type sufficiently overlaps with the other. If this was intentional, convert the expression to 'unknown' first. 

Type '{ fooBar: () => number; }' is missing the following properties from type 'FooBar': foo, bar
```

Let's revert to the expression:

```typescript
// The const isn't initialized, but let's ignore it
const b: FooBar

// TS error
const c = b as { fooBar: () => number }
```

Now, we'll check the error hint:

```
Conversion of type 'FooBar' to type '{ fooBar: () => number; }' may be a mistake because neither type sufficiently overlaps with the other. If this was intentional, convert the expression to 'unknown' first.

Types of property 'fooBar' are incompatible.  
Type '(value: string) => number' is not comparable to type '() => number'.  
Target signature provides too few arguments. Expected 1 or more, but got 0.
```

And this one is pointing to the issue we have - incompatible `fooBar` methods.

Understanding the logic behind the TypeScript typecasting mechanism can help us avoid unsafe casting in our code and keep it more reliable.

Use casts for good and good luck!