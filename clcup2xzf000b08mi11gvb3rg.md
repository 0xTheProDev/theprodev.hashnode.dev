# Building a Filter in a Complex React Application

Often time we have to build UIs that has filters to narrow down the content shown on the screen, The data point can be represented in tabular form or as a list whatsoever. However, as a User, it is hard to keep a track of what are the filters being applied.  
  
So quite several application renders a list of filters applied somewhere in the UI before the data part or at least within the filter component. That being said, we are not here to discuss UI challenges, but something more fundamental, states.  
  
How will you manage the state across multiple components? As it stands, the state is being consumed by at least three components, the actual list, the filter and the list of applied filters, and any change in the filter should affect all three.

## The Concerns

If we simplify the problem statement, we are left with mainly three concerns:

1. **The Data Part:** How will the Data Layer know what filters are being applied and produce results accordingly?
    
2. **The UI Part:** How can the user know and apply one or more filters in the UI?
    
3. **The Store Part:** How can we sync the filter information and persist throughout the lifetime of such UI?
    

## The Data Part

Based on how your *API* is designed, this can be achieved in multiple ways. In our case, we will assume we have *Query Parameters* over *GET* requests. Using native `fetch` we can implement the following:

```javascript
export const loadUsers = (filters) => fetch(`https://api.example.com/users?${new URLSearchParams(filters)}`);
```

## The UI Part

On the UI side, assuming we have the filter object, it is not much hard to pass them on to React Components and have the UI show which filters are available and which are already applied. I am keeping this part open-ended intentionally.

## The Store Part

Now that we have sorted out two of three concerns, here comes the most important part. We know if we pass our active filters to the `fetch` function we will get updated results. We also are sure that if we know the filters, we can show them in the UI.

Like how we have *Query Parameters* for the *API*, what about having the same for the page? After all, it is just a medium to the back-end data service!

So we store the filter object into the URL as Parameters post serialization. And deserialize during the render. This also gives us an edge for bookmarking and sharing preferable result sets. However, given that the URL can be modified outside the scope of the application, there needs to be validation logic as well.

Let's compile all of this into a reusable React hook:

```typescript
import { NextRouter, useRouter } from "next/router";
import * as React from "react";
import * as Yup from "yup";
import type Lazy from "yup/lib/Lazy";

/**
 * Available Options on Validation Schema.
 */
type Options<T extends Yup.AnyObjectSchema | Lazy<any>> = Parameters<
  T["validateSync"]
>[1];

/**
 * Query Filter Result Interface.
 */
type QueryFilterResult<T extends Record<string, any>> = {
  /**
   * Validation error on Filter Key Value pairs.
   */
  errors: Partial<T>;
  /**
   * Filter results using one or more Parameter.
   * @param filters - A Key Value pair of Filter Names and Values.
   */
  filterBy(filters: Partial<T>): Promise<void>;
  /**
   * Key Value pair of applicable Filters.
   */
  filters: T;
  /**
   * Reset Filters.
   */
  reset(filters?: Partial<T>): Promise<void>;
};

/**
 * Mutable Ref Object to hold current values of Filter and Validation Errors.
 */
type QueryFilterRef<T> = {
  /**
   * Validation error on Filter Key Value pairs.
   */
  errors: Partial<T>;
  /**
   * Key Value pair of applicable Filters.
   */
  values: T;
};

/**
 * Query Filter Interface.
 */
type QueryFilter = <T extends Yup.AnyObjectSchema | Lazy<any>>(
  /** Yup Schema of the Query Filters. */
  schema: T,
  /** Additional Schema Options for Validation. */
  options?: Options<T>,
) => QueryFilterResult<Yup.InferType<T>>;

/**
 * Push onto History Stack with Query Parameters.
 * @param router - Next Router Instance.
 * @param queryParams - Query Parameters.
 */
const pushStateWithQuery = <T extends Record<string, any>>(
  router: NextRouter,
  queryParams: T,
) => {
  const url = `${router.pathname}?${new URLSearchParams(queryParams)}`;
  return router.push(url, undefined, { scroll: false, shallow: true });
};

/**
 * Transforms Yup Validation Error to Serialsed Schema.
 * @param error - Yup Error Object.
 * @returns Seralised Error Object.
 */
const parseYupError = (error: Yup.ValidationError) => {
  if (!error.inner.length) {
    return {
      [error.path!]: error.errors[0],
    };
  }

  return (error.inner || []).reduce<Record<string, string>>(
    (previous, error) => {
      if (!previous[error.path!]) {
        previous[error.path!] = error.message;
      }

      return previous;
    },
    {},
  );
};

/**
 * Default Parameters for Schema Validation Options.
 */
const BaseSchemaOptions: Partial<Options<Yup.AnyObjectSchema | Lazy<any>>> = {
  abortEarly: true,
  stripUnknown: true,
} as const;

/**
 * Validates a given object against provided Schema and Schema Options.
 * @param object - Instance of Object that requires Validation.
 * @param schema - Schema to Validate against.
 * @param options - Schema Validation Options.
 * @returns An Object with Parsed Values and Validation Errors.
 */
const validateYupSchema = <
  T extends Yup.AnyObjectSchema | Lazy<any>,
  O extends Record<string, any>,
>(
  object: O,
  schema: T,
  options: Options<T>,
): QueryFilterRef<Yup.InferType<T>> => {
  try {
    return {
      values: schema.validateSync(
        object,
        Object.assign(BaseSchemaOptions, options),
      ),
      errors: {},
    };
  } catch (error) {
    return {
      values: {},
      errors: parseYupError(error as Yup.ValidationError) as Partial<
        Yup.InferType<T>
      >,
    };
  }
};

/**
 * Create Filters out of Query Parameters in current Page.
 * @param schema - Yup Schema of the Query Filters.
 * @param options - Additional Schema Options for Validation.
 * @returns Parsed Filters, callback to update Filters and Error Object in case of validation error.
 */
export const useQueryFilter: QueryFilter = (schema, options) => {
  type Schema = Yup.InferType<typeof schema>;

  const router = useRouter();

  const filtersRef = React.useRef<QueryFilterRef<Schema>>({
    values: {},
    errors: {},
  });

  filtersRef.current = validateYupSchema(router.query, schema, options);

  /** Cleanup Query Parameters */
  useInsertionEffect(() => {
    const { values: filters } = filtersRef.current;
    pushStateWithQuery(router, filters);
  }, []);

  const filterBy = React.useCallback(async (filters: Partial<Schema>) => {
    /** This avoids Stale Closure problem by using Mutable Ref instance. */
    const { values: _filters } = filtersRef.current;
    const nextFilters = {
      ..._filters,
      ...filters,
    };

    const pushStatePromise = pushStateWithQuery(router, nextFilters);

    /** This ensures consecutive call of `filterBy` does not reset earlier filters. */
    filtersRef.current = {
      errors: {},
      values: nextFilters,
    };

    /** This is useful for chaining promises in consumer component or hook. */
    await pushStatePromise;
  });

  const reset = React.useCallback(async (filters: Partial<Schema> = {}) => {
    const pushStatePromise = pushStateWithQuery(router, filters);

    /** This ensures subsequent call of `filterBy` does not reuse earlier filters. */
    filtersRef.current = {
      errors: {},
      values: filters,
    };

    /** This is useful for chaining promises in consumer component or hook. */
    await pushStatePromise;
  });

  const { values: filters, errors } = filtersRef.current;

  return {
    errors,
    filterBy,
    filters,
    reset,
  };
};
```

That is it. Now you have a React hook managing your state, and on the consumer side, you can use the current state to manage and update the result set and show them in your UI.