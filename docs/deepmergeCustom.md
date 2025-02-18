# Deepmerge Custom

`deepmergeCustom` allows you to customize the deepmerge function. It is a higher-order function; that is to say it returns a new customized deepmerge function.

## Merge Functions

The signature of merging functions looks like this:

```ts
(values: Ts, utils: U, meta: M | undefined) => unknown;
```

Simply return the merged value.

### Special Actions

We provide a couple of special actions under `utils.actions` that you can use to simplify your custom merge functions.

#### Skipping a Property (`utils.actions.skip`)

If you want to skip a property from being included in the result of a merge based on its value or metadata, you can easily do so with this action.

For example, skipping all properties under the key `"skipme"` of type `Date`:

```ts
const customizedDeepmerge = deepmergeCustom({
  mergeOthers: (value, utils, meta) => {
    if (meta?.key === "skipme") {
      const nonDateValues = values.filter((value) => !(value instanceof Date));
      if (nonDateValues.length === 0) {
        return utils.actions.skip; // Completely skip this property
      }

      // Don't skip the property completely if a non-Date value was found.
      return utils.defaultMergeFunctions.mergeOthers(nonDateValues);
    }

    // Perform the default merging (see below).
    return utils.actions.defaultMerge;
  },
});
```

To do this without any special actions would require using a custom `mergeRecords` which would be a bit more complicated.

#### Default Merging (`utils.actions.defaultMerge`)

If you do not want to have to explicitly call the default merging function in your custom merge function;
you can just return `utils.actions.defaultMerge`. This will automatically apply the default merging strategy.

For example, the following `customizedDeepmerge` functions are equivalent:

```ts
const customizedDeepmerge = deepmergeCustom({
  mergeOthers: (value, utils) => {
    if (someCondition) {
      return someCustomValue;
    }
    return utils.defaultMergeFunctions.mergeOthers(values);
  },
});
```

```ts
const customizedDeepmerge = deepmergeCustom({
  mergeOthers: (value, utils) => {
    if (someCondition) {
      return someCustomValue;
    }
    return utils.actions.defaultMerge;
  },
});
```

Note: When using this action, you cannot change the values upon which the default merging will apply.

##### Implicit Default Merging

You can alternatively set the option `enableImplicitDefaultMerging` to `true` to make it so that if any of your
custom merge functions return `undefined`, then the default merging strategy will automatically be applied.

For example, the following `customizedDeepmerge` function is equivalent to the two above:

```ts
const customizedDeepmerge = deepmergeCustom({
  enableImplicitDefaultMerging: true,  // enable implicit default merging
  mergeOthers: (value, utils) => {
    if (someCondition) {
      return someCustomValue;
    }
    // implicitly return undefined
  },
});
```

## Customizing the Return Type

If you want to customize the deepmerge function, you probably also want the return type of the result to be correct too.\
Unfortunately however, due to TypeScript limitations, we can not automatically infer this.
In order to get the correct return type, you need to provide us with type information about how you have customized the function (we do the very same to define the default configuration).

We need to use HKTs (higher-kinded types) in order to generate the right output type. But again, unfortunately, TypeScript does not support HKTs. Luckily however, there is a workaround.
To use HKTs, we alias the type to a string type (a URI) and simply refer to that type by its alias until we need to resolve it.

Here's a simple example that creates a custom deepmerge function that does not merge arrays.

```js
import type { DeepMergeLeafURI } from "deepmerge-ts";
import { deepmergeCustom } from "deepmerge-ts";

const customDeepmerge = deepmergeCustom<{
  DeepMergeArraysURI: DeepMergeLeafURI; // <-- Needed for correct output type.
}>({
  mergeArrays: false,
});

const x = { foo: [1, 2], bar: [3, 4] };
const y = { foo: [5, 6] };

customDeepmerge(x, y); // => { foo: [5, 6], bar: [3, 4] }
```

When resolving a HKT, we use a lookup inside an interface called `DeepMergeMergeFunctionURItoKind`.
This interface needs to contain all the mappings of the URIs to their actual type.

When defining your own HKT for use with deepmerge, you need to extend this interface with your mapping.
This can be done using [Declaration Merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html) by declaring a module block for this library and defining the same interface.

```ts
declare module "deepmerge-ts" {
  interface DeepMergeMergeFunctionURItoKind<Ts extends ReadonlyArray<unknown>, MF extends DeepMergeMergeFunctionsURIs, M> {
    readonly MyCustomMergeURI: MyValue;
  }
}
```

Here's an example of creating a custom deepmerge function that amalgamates dates into an array.

```ts
import type { DeepMergeLeaf, DeepMergeMergeFunctionURItoKind, DeepMergeMergeFunctionsURIs } from "deepmerge-ts";
import { deepmergeCustom } from "deepmerge-ts";

const customizedDeepmerge = deepmergeCustom<{
  DeepMergeOthersURI: "MyDeepMergeDatesURI"; // <-- Needed for correct output type.
}>({
  mergeOthers: (values, utils, meta) => {
    // If every value is a date, the return the amalgamated array.
    if (values.every((value) => value instanceof Date)) {
      return values;
    }
    // Otherwise, use the default merging strategy.
    return utils.defaultMergeFunctions.mergeOthers(values);
  },
});

const x = { foo: new Date("2020-01-01") };
const y = { foo: new Date("2021-02-02") };
const z = { foo: new Date("2022-03-03") };

customDeepmerge(x, y, z); // => { foo: [Date, Date, Date] }

declare module "deepmerge-ts" {
  interface DeepMergeMergeFunctionURItoKind<
    Ts extends ReadonlyArray<unknown>,
    MF extends DeepMergeMergeFunctionsURIs,
    M
  > {
    readonly MyDeepMergeDatesURI: EveryIsDate<Ts> extends true ? Ts : DeepMergeLeaf<Ts>;
  }
}

type EveryIsDate<Ts extends ReadonlyArray<unknown>> = Ts extends readonly [infer Head, ...infer Rest]
  ? Head extends Date
    ? EveryIsDate<Rest>
    : false
  : true;
```

Note: If you want to use HKTs in your own project, not related to deepmerge-ts, we recommend checking out [fp-ts](https://gcanti.github.io/fp-ts/modules/HKT.ts.html).

## Meta Data

We provide a simple object of meta data that states the key that the values being merged were under.

Here's an example that creates a custom deepmerge function that merges numbers differently based on the key they were under.

```ts
import type { DeepMergeLeaf, DeepMergeMergeFunctionURItoKind, DeepMergeMergeFunctionsURIs } from "deepmerge-ts";
import { deepmergeCustom } from "deepmerge-ts";

const customizedDeepmerge = deepmergeCustom({
  mergeOthers: (values, utils, meta) => {
    if (meta !== undefined && areAllNumbers(values)) {
      const { key } = meta;
      const numbers: ReadonlyArray<number> = values;

      if (key === "sum") {
        return numbers.reduce((sum, value) => sum + value);
      }
      if (key === "product") {
        return numbers.reduce((prod, value) => prod * value);
      }
      if (key === "mean") {
        return numbers.reduce((sum, value) => sum + value) / numbers.length;
      }
    }

    return utils.defaultMergeFunctions.mergeOthers(values);
  },
});

function areAllNumbers(values: ReadonlyArray<unknown>): values is ReadonlyArray<number> {
  return values.every((value) => typeof value === "number");
}

const v = { sum: 1, product: 2, mean: 3 };
const x = { sum: 4, product: 5, mean: 6 };
const y = { sum: 7, product: 8, mean: 9 };
const z = { sum: 10, product: 11, mean: 12 };

customizedDeepmerge(v, x, y, z); // => { sum: 22, product: 880, mean: 7.5 }
```

### Customizing the Meta Data

You can customize the meta data that is passed to the merge functions by providing a `metaDataUpdater` function.

Here's an example that uses custom metadata that accumulates the full key path.

```ts
import type { DeepMergeLeaf, DeepMergeMergeFunctionURItoKind, DeepMergeMergeFunctionsURIs } from "deepmerge-ts";
import { deepmergeCustom } from "deepmerge-ts";

const customizedDeepmerge = deepmergeCustom<
  // Change the return type of `mergeOthers`.
  {
    DeepMergeOthersURI: "KeyPathBasedMerge";
  },
  // Change the meta data type.
  {
    keyPath: ReadonlyArray<PropertyKey>;
  }
>({
  // Customize what the actual meta data.
  metaDataUpdater: (previousMeta, metaMeta) => {
    if (previousMeta === undefined) {
      if (metaMeta.key === undefined) {
        return { keyPath: [] };
      }
      return { keyPath: [metaMeta.key] };
    }
    if (metaMeta.key === undefined) {
      return previousMeta;
    }
    return {
      ...metaMeta,
      keyPath: [...previousMeta.keyPath, metaMeta.key],
    };
  },
  // Use the meta data when merging others.
  mergeOthers: (values, utils, meta) => {
    if (
      meta !== undefined &&
      meta.keyPath.length >= 2 &&
      meta.keyPath[meta.keyPath.length - 2] === "bar" &&
      meta.keyPath[meta.keyPath.length - 1] === "baz"
    ) {
      return "special merge";
    }

    return utils.defaultMergeFunctions.mergeOthers(values);
  },
});

const x = {
  foo: { bar: { baz: 1, qux: 2 } },
  bar: { baz: 3, qux: 4 },
};
const y = {
  foo: { bar: { baz: 5, bar: { baz: 6, qux: 7 } } },
  bar: { baz: 8, qux: 9 },
};

customizedDeepmerge(x, y); // => { foo: { bar: { baz: "special merge", bar: { baz: 6, qux: 7 }, qux: 2 } }, bar: { baz: "special merge", qux: 9 }, }

declare module "../src/types" {
  interface DeepMergeMergeFunctionURItoKind<
    Ts extends Readonly<ReadonlyArray<unknown>>,
    MF extends DeepMergeMergeFunctionsURIs,
    M // This is the meta data type
  > {
    readonly KeyPathBasedMerge: Ts[number] extends number
      ? Ts[number] | string
      : DeepMergeLeaf<Ts>;
  }
}
```

## API

See [deepmerge custom API](./API.md#deepmergecustomoptions-rootmetadata).

# Deepmerge Into Custom

`deepmergeIntoCustom` as the name suggests, works just like `deepmergeCustom`, only for `deepmergeInto` instead of `deepmerge`.
But there are some differences to be aware of.

## Merge Functions

The signature of merging functions for `deepmergeIntoCustom` looks like this:

```ts
(target: DeepMergeValueReference<T>, values: Ts, utils: U, meta: M | undefined) => void | symbol;
```

Instead of returning a value like with `deepmergeCustom`'s merge functions, mutations should be made to `target.value`.\
You can however still return an action.

Note: `values` includes all the values, including the target's value (if there is one).

### Special Actions

#### No Skip Action (`utils.actions.skip`)

This action doesn't make sense with in the context of merging into a target.
Use `delete target.value[key]` instead if you don't want the property to exists on the target.

#### No Implicit Default Merging

It doesn't make sense to have implicit default merging here as all merge functions should return `undefined` (if not returning an action).

## Customizing the Return Type

The return type of a custom `deepmergeInto` should be void, so you don't need to customize it's return type like you would with a regular custom `deepmerge` function.

However, you may want to use an assertion function if the target's type is not the same as the inputs.
This is by no means required though.
But if you want to do this then you'll simply need to explicity declare a type annotation for your customized `deepmergeInto` function that makes such an assertion.

Here's an example:

```ts
type CustomizedDeepmergeInto = <
  Target extends object,
  Ts extends ReadonlyArray<object>
>(
  target: Target,
  ...objects: Ts
) => asserts target is Target & // Unioning with `Target` is essentially required to make TypeScript happy.
  DeepMergeHKT<
    [Target, ...Ts], // Don't forget to pass the `Target` type here too.
    {
      DeepMergeRecordsURI: DeepMergeMergeFunctionsDefaultURIs["DeepMergeRecordsURI"]; // Use default behavior.
      DeepMergeArraysURI: DeepMergeMergeFunctionsDefaultURIs["DeepMergeArraysURI"]; // Use default behavior.
      DeepMergeSetsURI: DeepMergeMergeFunctionsDefaultURIs["DeepMergeSetsURI"]; // Use default behavior.
      DeepMergeMapsURI: DeepMergeMergeFunctionsDefaultURIs["DeepMergeMapsURI"]; // Use default behavior.
      DeepMergeOthersURI: "CustomDeepMergeOthersURI"; // Use custom behavior (see deepmergeCustom's docs above for details).
    },
    DeepMergeBuiltInMetaData // Use default meta data.
  >;

export const customizedDeepmergeInto: CustomizedDeepmergeInto =
  deepmergeIntoCustom({
    mergeOthers: (source, values, utils, meta) => {
      /* ... */
    },
  });
```

## API

See [deepmerge into custom API](./API.md#deepmergeintocustomoptions-rootmetadata).
