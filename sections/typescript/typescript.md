# Typescript Coding Standards

## Interface vs Type Alias 

TypeScript supports [type aliases](https://www.typescriptlang.org/docs/handbook/advanced-types.html#type-aliases) for naming a type expression. This can be used to name primitives, unions, tuples, and any other types. 

However, when declaring types for objects, use interfaces instead of a type alias for the object literal expression.

These forms are nearly equivalent, so under the principle of just choosing one out of two forms to prevent variation, we should choose one. Additionally, there also [interesting technical reasons to prefer interface](https://ncjamieson.com/prefer-interfaces/). That page quotes the TypeScript team lead: Honestly, my take is that it should really just be interfaces for anything that they can model. There is no benefit to type aliases when there are so many issues around display/perf.

Example: 

✅
```
interface User {
  firstName: string;
  lastName: string;
}
```
❌
```
type User = {
  firstName: string,
  lastName: string,
}
```