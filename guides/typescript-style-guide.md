Typescript Style Guide
======================
### `"strict": true` must be enabled in tsconfig.json
* This makes Typescript much safer, and saves us from countless hard-to-debug errors

### Use `===`/`!==` rather than `==`/`!=`
Double-equal (`==`/`!=`) behaves strangely in Javascript, and won't always do what you want

### Default to `for... of` (iterate over values) rather than `for... in` (iterate over keys)
* `for... in` will sometimes do unintuitive things
    * E.g. `for... in` on an array will yield array indexes - `0`, `1`, `2`, etc.

### Constants should be `UPPER_SNAKE_CASE`d
* Makes very clear that these shouldn't be changed

### Class properties should be `private readonly` wherever possible
* Rationale: reduces mutability which makes things easier to debug & reason about

### Helper functions should be `private` and `static` wherever possible
* Rationale: reduces

### For returning errors, use the `Result` class from the `neverthrow` package
* Rationale: makes it explicit when a function can return an error result (in contrast to exceptions)

### When a function returns only an `Error`, make the return type `Result<null, Error>`
* Rationale: gives the option to return an `Error`, and makes it explicit that the user should check the error

### Do not use `ResultAsync`, `okAsync`, or `errAsync` from the `neverthrow` package
* Rationale: `async` functions MUST return `Promise<...>`, and `ResultAsync<..>` doesn't fulfill that contract

### Wherever possible, use `async`/`await` code
* Rationale: `async`/`await` code is much easier to read than callback code

### Use `import type` rather than `import`
`import type` appears to be strictly better than `import`, [guarding against several edge cases and allowing for extra optimization](https://stackoverflow.com/questions/61412000/do-i-need-to-use-the-import-type-feature-of-typescript-3-8-if-all-of-my-import).

NOTE: We used to use `import` until we discovered that `import type` was better, so any `import` references should be switched to `import type`.

### Prefer parameter property class declaration when possible
Typescript has neat syntactic sugar to combine the constructor and property declaration of a class into a condensed package, [called "parameter property" style](https://google.github.io/styleguide/tsguide.html#parameter-property-comments):

```typescript
class Pizza {
    constructor(
        private readonly crust: Crust,
        private readonly toppings: Topping[],
    ) {}
}
```

We prefer this over the more verbose, Java-like style.
