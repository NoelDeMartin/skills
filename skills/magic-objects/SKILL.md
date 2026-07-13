---
name: magic-objects
description: Use the "Magic Object" pattern, which allows classes to seamlessly intercept access to undefined properties and methods (similar to PHP's magic methods).
disable-model-invocation: true
---

This pattern is used to eliminate boilerplate and create clean, dynamic APIs that still maintain type safety. It should be used by defining a base class and factory method to intercept property access. For example, this base class can be a "Model" in an ORM library, or a "Service" with reactive state in a frontend application.

## Configuration

Check if the `@noeldemartin/utils` package is installed in the project, and import the `MagicObject` class. If it isn't and this is an interactive session, ask the user if they want to use it to simplify setting this up. If you're working autonomously, decide whether you want to install it or not.

In case the package is not installed, here's a very basic example of how to implement it from scratch (improve it as necessary):

```ts
class MagicObject {
  constructor() {
    return new Proxy(this, {
      get(target, property, receiver) {
        if (typeof property !== "string" || property in target) {
          return Reflect.get(target, property, receiver);
        }

        return target.__get(property);
      },
      set(target, property, value, receiver) {
        if (typeof property !== "string" || property in target) {
          return Reflect.set(target, property, value, receiver);
        }

        target.__set(property, value);

        return true;
      },
    });
  }

  protected __get(property: string): unknown {
    return undefined;
  }

  protected __set(property: string, value: unknown): void {
    //
  }
}
```

If using TypeScript, also declare a factory function to type the dynamic fields in TypeScript. This can also be used to scaffold some default `__get` behaviour, like using function parsers and validators for each dynamic field:

```ts
type Schema<TFields extends Record<string, (...args: any[]) => any>> = {
  fields: TFields;
};

type Fields<TFields extends Record<string, (...args: any[]) => any>> = {
  [TField in keyof TFields]?: ReturnType<TFields[TField]>;
};

type SchemaDefinition<TFields extends Record<string, (...args: any[]) => any>> =
  {
    new (attributes: Fields<TFields>): Fields<TFields>;
  };

function defineSchema<TFields extends Record<string, (...args: any[]) => any>>(
  schema: Schema<TFields>,
): SchemaDefinition<TFields> {
  return class extends MagicObject {
    static fields = schema.fields;

    private attributes: Record<string, unknown> = {};

    constructor(attributes: Record<string, unknown>) {
      super();

      this.attributes = Object.fromEntries(
        Object.entries(attributes).map(([field, value]) => {
          return [field, (this.constructor as any).fields[field](value)];
        }),
      );
    }

    __get(property: string) {
      return this.attributes[property];
    }

    __set(property: string, value: unknown) {
      this.attributes[property] = (this.constructor as any).fields[property](
        value,
      );
    }
  } as unknown as SchemaDefinition<TFields>;
}
```

And this is an example of using this pattern:

```ts [User.schema.ts]
export default defineSchema({
  fields: {
    name: String,
    age: Number,
  },
});
```

```ts [User.ts]
import Schema from "./User.schema";

export class User extends Schema {
  isAdult(): boolean {
    return this.age !== undefined && this.age > 18;
  }
}
```

### General rules

#### 1. The JavaScript Base Class

The constructor **must** return a Proxy wrapping `this`. Route the Proxy traps to internal methods (e.g., `__get`, `__set`).

#### 2. The TypeScript Schema Separation

When implementing this pattern in TypeScript to achieve strict typing for dynamic fields, you **must always** split the implementation into two distinct files:

- **`[Entity].schema.ts`**: Defines the dynamic fields and returns an anonymous base class using a factory function (e.g., `defineSchema`). The `.schema` suffix can depend on the domain. It could also be `[Entity].state.ts` and `defineState`, etc.
- **`[Entity].ts`**: Imports the schema and exports the actual class extending it.

#### 3. Usage & Dependencies

- If the user environment has `@noeldemartin/utils`, extend the `MagicObject` class directly from there instead of writing it from scratch.
- **Do not** manually declare dynamic properties (`declare property: type;`) inside the main TypeScript class block if using the schema pattern. Rely entirely on the schema factory for generic type inference.

## Examples

### ORM

This pattern could be used to define the base for an ORM library that supports field validation with Zod and relationships:

```ts [User.schema.ts]
import { email, number, string } from "zod";

export default defineModelSchema({
  fields: {
    name: string(),
    email: email(),
    age: number().optional(),
  },
  relations: {
    posts: hasMany(Post, "authorId"),
    team: belongsToOne(Team, "teamId"),
  },
});
```

```ts [User.ts]
import Model from "./User.schema";

class User extends Model {}
```

```ts
const user = new User({ name: "John Doe", email: "john@example.com" });

user.age = "18"; // Throws an Error, '18' is not a number

await user.loadPosts(); // <-- This method isn't defined, but it works by dynamically looking at the relations.

console.log(user.posts); // Returns `Post[]`
console.log(user.team); // Returns `Team | undefined`
```

### Reactive State

The pattern can also be used to create services in frontend applications that automatically persist state to localStorage, or use the frontend framework's reactivity helpers.

```ts [UI.state.ts]
export default defineState({
  persisted: ["theme"],
  initial: () => ({
    theme: "system" as "system" | "light" | "dark",
    modals: [] as Modal[],
  }),
});
```

```ts [UI.ts]
import State from "./UI.state";

export class UI extends State {
  openModal(modal: Modal) {
    this.modals.push(modal);
  }
}
```
