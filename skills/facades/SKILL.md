---
name: facades
description: Use the "Facade" pattern, which exposes global service instances through static-like accessors that can be swapped at runtime.
disable-model-invocation: true
---

This pattern is used to access shared services globally without passing instances around (like singletons), while still being able to swap the underlying instance at runtime (like dependency injection). This makes the code simpler to write and read, and easy to test.

Use it for application-wide services: integrations with 3rd party APIs (payments, analytics, storage), shared state in frontend applications, expensive or lazily loaded dependencies, etc. It can also be used just to wrap native functionality that you may want to mock in tests, such as HTTP requests (`fetch`) or interacting with storage (`localStorage`, `IndexedDB`).

Do not use it for objects that have multiple instances or carry per-request/per-entity data; those should still be passed around explicitly.

## Configuration

Check if the `@noeldemartin/utils` package is installed in the project, and import the `facade` function. If it isn't and this is an interactive session, ask the user if they want to use it to simplify setting this up. If you're working autonomously, decide whether you want to install it or not.

In case the package is not installed, here's a very basic example of how to implement it from scratch (improve it as necessary):

```ts
export type Facade<T extends object> = T & {
  setInstance(instance: T): T;
  reset(): T;
};

export function facade<T extends object>(
  classOrFactory: (() => T) | (new () => T),
): Facade<T> {
  let instance: T | null = null;

  const newInstance = (): T => {
    try {
      return (classOrFactory as () => T)();
    } catch {
      return new (classOrFactory as new () => T)();
    }
  };

  const getInstance = (): T => (instance ??= newInstance());

  const methods = {
    setInstance: (newValue: T): T => (instance = newValue),
    reset: (): T => (instance = newInstance()),
  };

  return new Proxy(methods as Facade<T>, {
    get: (_, property) =>
      property in methods
        ? Reflect.get(methods, property)
        : Reflect.get(getInstance(), property),
    set: (_, property, value) => Reflect.set(getInstance(), property, value),
  });
}
```

And this is an example of using this pattern:

```ts [PaymentGateway.ts]
export interface IPaymentGateway {
  charge(amount: number): Promise<void>;
}

export class StripePaymentGateway implements IPaymentGateway {
  async charge(amount: number): Promise<void> {
    // ...
  }
}

export default facade<IPaymentGateway>(StripePaymentGateway);
```

```ts [ShoppingCart.ts]
import PaymentGateway from "./PaymentGateway";

export class ShoppingCart {
  constructor(private price: number) {}

  async purchase(): Promise<void> {
    await PaymentGateway.charge(this.price);
  }
}
```

```ts [ShoppingCart.test.ts]
import PaymentGateway from "./PaymentGateway";
import { ShoppingCart } from "./ShoppingCart";

it("makes a purchase", async () => {
  const cart = new ShoppingCart(42);
  const paymentMock = { charge: vi.fn() };

  PaymentGateway.setInstance(paymentMock);

  await cart.purchase();

  expect(paymentMock.charge).toHaveBeenCalledWith(42);
});
```

### General rules

#### 1. Naming

- The facade is exported as the default export of the module, using the name of the service without suffixes (e.g., `PaymentGateway`, `Workspaces`).
- The underlying class uses a suffix or prefix that describes it (e.g., `WorkspacesService`, `StripePaymentGateway`), and the contract (if any) is an interface (e.g., `IPaymentGateway`).
- If the facade has a default implementation and alternative ones, type the facade with the interface (`facade<IPaymentGateway>(...)`) so that any implementation can be swapped in.

#### 2. Usage

- Call facades directly where they are needed. **Do not** inject facades through constructors or function arguments; that defeats the purpose of the pattern.
- **Do not** keep references to the underlying instance or its methods (e.g., `const charge = PaymentGateway.charge`) in long-lived variables, because they won't be updated if the instance is swapped.
- Facades are global state. Don't create facades for things that aren't conceptually global to the application.

#### 3. Testing

- Swap instances with `setInstance()` (or a mocking helper, if the project has one) instead of mocking modules.
- Make sure to restore facades between tests (e.g., calling `reset()` in an `afterEach` hook), so that mocks don't leak across tests.
- If a service interacts with a 3rd party API, consider configuring a default mock for the entire test suite in the global test setup.

## Lazy facades

Facades only resolve their instance when they're first used. This can be taken further to defer async work, such as code splitting or async initialization, using "lazy facades". In this case, all the methods become async (even if they weren't in the original class), and properties must be read asynchronously with `getProperty()`.

If `@noeldemartin/utils` is installed, import the `lazyFacade` function from there. Besides the instance methods, lazy facades have the following methods:

- `getInstance()`: Returns a Promise with the underlying instance, creating it if necessary.
- `getProperty(property)`: Returns a Promise with the value of a property in the underlying instance.
- `setInstance(instance)`: Swaps the underlying instance (even if the factory is still loading).
- `reset()`: Discards the underlying instance, so that the factory is called again the next time a method is used.

Use `getInstance()` when you need to read or write many properties, instead of calling `getProperty()` repeatedly. And keep in mind that a lazy facade is not a regular facade, so it can't be used with `isFacade()` or `getFacadeInstance()` (use `isLazyFacade()` instead).

In case the package is not installed, here's a very basic example of how to implement it from scratch (improve it as necessary):

```ts
export type LazyFacade<T> = {
  [K in keyof T]: T[K] extends (...args: infer TArgs) => infer TResult
    ? (...args: TArgs) => Promise<Awaited<TResult>>
    : never;
};

export function lazyFacade<T extends object>(
  factory: () => Promise<T>,
): LazyFacade<T> {
  let instance: Promise<T> | null = null;

  const getInstance = (): Promise<T> =>
    (instance ??= factory().catch((error) => {
      instance = null;

      throw error;
    }));

  return new Proxy({} as LazyFacade<T>, {
    get: (_, property) => {
      // Prevent the facade from being treated as a thenable by `await`.
      if (property === "then") {
        return undefined;
      }

      return async (...args: unknown[]) => {
        const target = await getInstance();
        const method = Reflect.get(target, property) as (
          ...args: unknown[]
        ) => unknown;

        return method.apply(target, args);
      };
    },
  });
}
```

This is how they can be used for code splitting:

```ts
const PaymentGateway = lazyFacade(async () => {
  const { default: payment } = await import("./my/payment/gateway");

  return payment;
});

// The chunk is only loaded when the first method is called.
await PaymentGateway.charge(42);
```

And this is how they can be used for async initialization, avoiding the need to check if the object has been initialized everywhere in the codebase:

```ts
const Expensive = lazyFacade(async () => {
  const instance = new ExpensiveService();

  await instance.initialize();

  return instance;
});

await Expensive.doSomething(42);
```

## Examples

### Services

In frontend applications, this pattern can be combined with the "Magic Objects" pattern to create services that encapsulate shared business logic and reactive state, available anywhere in the application:

```ts [Workspaces.ts]
import State from "./Workspaces.state";

export class WorkspacesService extends State {
  async delete(workspace: Workspace): Promise<void> {
    // ...
  }
}

export default facade(WorkspacesService);
```

```ts
import Workspaces from "@/services/Workspaces";

await Workspaces.delete(workspace);
```

### Code splitting

Lazy facades can be used to keep heavy dependencies out of the main bundle. For example, a service that exports reports as PDF files using a large library:

```ts [PdfExportService.ts]
import { jsPDF } from "jspdf";

export class PdfExportService {
  download(report: Report): void {
    const pdf = new jsPDF();

    pdf.text(report.title, 10, 10);
    pdf.save(`${report.slug}.pdf`);
  }
}
```

```ts [PdfExport.ts]
import type { PdfExportService } from "./PdfExportService";

export default lazyFacade<PdfExportService>(async () => {
  const { PdfExportService } = await import("./PdfExportService");

  return new PdfExportService();
});
```

```ts [ReportPage.ts]
import PdfExport from "./PdfExport";

downloadButton.addEventListener("click", async () => {
  await PdfExport.download(report);
});
```

Importing `PdfExport` is cheap, because it only contains the facade. The chunk with `PdfExportService` (and `jspdf`) is not downloaded when the page loads, or when `PdfExport` is imported; it's only downloaded the first time a method is called (in this case, when the user clicks the button). After that, the same instance is reused, and if several methods are called while the chunk is loading, it is only downloaded once.

For this to work, the implementation module **must only** be imported dynamically. Other modules can reference its types with `import type`, which is always erased during compilation, but importing any value from it statically (such as the class) will add the module to the main bundle.

If you want to start downloading the chunk before it's needed (for example, when the user hovers the button), call `PdfExport.getInstance()` without awaiting it. And in tests, calling `PdfExport.setInstance()` with a mock before using the facade means the chunk will never be loaded.

### Wrapping native APIs

Facades can also wrap native functionality in order to mock it in tests. For example, this is how to wrap `localStorage`. The service accesses the native API through a protected getter, so that alternative implementations only need to override that getter:

```ts [Storage.ts]
export class StorageService {
  set<T>(key: string, value: T): void {
    this.storage.setItem(key, JSON.stringify(value));
  }

  get<T>(key: string): T | null {
    const value = this.storage.getItem(key);

    return value === null ? null : JSON.parse(value);
  }

  remove(key: string): void {
    this.storage.removeItem(key);
  }

  protected get storage(): Pick<Storage, "getItem" | "setItem" | "removeItem"> {
    return localStorage;
  }
}

export default facade(StorageService);
```

```ts [InMemoryStorageService.ts]
import { StorageService } from "./Storage";

export default class InMemoryStorageService extends StorageService {
  readonly items = new Map<string, string>();

  protected override get storage(): Pick<
    Storage,
    "getItem" | "setItem" | "removeItem"
  > {
    return {
      getItem: (key) => this.items.get(key) ?? null,
      setItem: (key, value) => void this.items.set(key, value),
      removeItem: (key) => void this.items.delete(key),
    };
  }
}
```

```ts [Preferences.test.ts]
import InMemoryStorageService from "./InMemoryStorageService";
import Storage from "./Storage";
import { saveTheme } from "./Preferences";

describe("Preferences", () => {
  let storage: InMemoryStorageService;

  beforeEach(() => {
    storage = new InMemoryStorageService();
    Storage.setInstance(storage);
  });

  it("saves the theme", () => {
    saveTheme("dark");

    expect(storage.items.get("theme")).toEqual('"dark"');
  });
});
```

The `@noeldemartin/utils` package already includes the `Storage` facade and `InMemoryStorageService` class, so use them instead if the package is installed. The same approach can be followed with other native APIs, for example wrapping `fetch` in an `Http` facade.
