# Graph

The [graph module](https://github.com/gchq/Gaffer/tree/master/core/graph) contains the Gaffer `Graph` object and related utilities. This is the entry point (or proxy) for your chosen Gaffer store.

The `Graph` separates the user from the underlying store. It holds a connection which acts as a proxy, delegating operations to the store.
It provides users with a single point of entry for executing operations on a store. This allows the underlying store to be swapped and the same operations can still be applied.

When you instantiate a `Graph`, this doesn't mean you are creating an entirely new graph with its own data, you are simply creating a connection to a store where some data is held.

See the [Graph Javadoc](https://gchq.github.io/Gaffer/uk/gov/gchq/gaffer/graph/package-summary.html) for further documentation.

## Creating a Graph

To create an instance of `Graph`, we recommend you use the `Graph.Builder` class. This has several helpful methods to create the graph from various different sources.
Most of the time, a graph requires just 3 things: some store properties, a schema and some graph specific configuration.

An example of creating a basic `Graph` object:

```java
Graph graph = new Graph.Builder()
      .config(new GraphConfig.Builder()
            .graphId("uniqueNameOfYourGraph")
            .build())
      .addSchemas(schemas)
      .storeProperties(storeProperties)
      .build();
```

Instead of a store properties, a store can be passed in directly. This is the easiest way to configure schema-less stores (Proxy and Federated Stores) using Java. See the [Java section of the Proxy Store reference page for an example](../../../administration-guide/gaffer-stores/proxy-store.md#using-a-proxystore-from-java). Using a Proxy Store will allow for connecting to an existing remote graph through Java, without needing to use the REST API or JSON directly.

## Store Properties
The store properties tells the graph the type of store to connect to along with any required connection details. See the [Stores](../../../administration-guide/gaffer-stores/store-guide.md) reference page for more information on the different Stores for Gaffer.

## Schema
The schema is passed to the store to instruct the store how to store and process the data.
See [Schemas](../../../administration-guide/gaffer-config/schema.md) for detailed information on schemas and the [Java API section of that page](../../../administration-guide/gaffer-config/schema.md#java-api) for lower level info.

## Graph Configuration
The graph configuration allows you to apply special customisations to the Graph instance. The only required field is the `graphId`.

To create an instance of `GraphConfig` you can use the `GraphConfig.Builder` class, or create it using a json file.

The `GraphConfig` can be configured with the following:

 - `graphId` - The `graphId` is a String field that uniquely identifies a `Graph`. When backed by a Store like Accumulo, this `graphId` is used as the name of the Accumulo table for the `Graph`.
 - `description` - a string describing the `Graph`.
 - `view` - The `Graph View` allows a graph to be configured to only return a subset of Elements when any Operation is executed. For example if you want your `Graph` to only show data that has a count more than 10 you could add a View to every operation you execute, or you can use this `Graph View` to apply the filter once and it would be merged into to all Operation Views so users only ever see this particular view of the data.
 - `library` - This contains information about the `Schema` and `StoreProperties` to be used.
 - `hooks` - A list of `GraphHook`s that will be triggered before, after and on failure when operations are executed on the `Graph`. See [GraphHooks](#graph-hooks) for more information.

Here is an example of a `GraphConfig`:

```java
new GraphConfig.Builder()
    .config(new GraphConfig.Builder()
            .graphId("exampleGraphId")
            .build())
    .description("Example Graph description")
    .view(new View.Builder()
            .globalElements(new GlobalViewElementDefinition.Builder()
                    .postAggregationFilter(new ElementFilter.Builder()
                        .select("ExamplePropertyName")
                        .execute(new IsLessThan("10"))
                        .build())
                    .build())
            .build())
    .library(new FileGraphLibrary())
    .addHook(new Log4jLogger())
  .build();
```

and in json:

```json
{
  "graphId": "exampleGraphId",
  "description": "Example Graph description",
  "view": {
    "globalElements": [
      {
        "postAggregationFilterFunctions": [
          {
            "predicate": {
              "class": "uk.gov.gchq.koryphe.impl.predicate.IsLessThan",
              "orEqualTo": false,
              "value": "10"
            },
            "selection": ["ExamplePropertyName"]
          }
        ]
      }
    ]
  },
  "library": {
      "class": "uk.gov.gchq.gaffer.store.library.FileGraphLibrary"
  },
  "hooks": [
    {
      "class": "uk.gov.gchq.gaffer.graph.hook.Log4jLogger"
    }
  ]
}
```

## Graph Hooks
The `Graph` class is final and must be used when creating a new connection to a store. We want to ensure that all users have a common point of entry to Gaffer, so all users have to start by instantiating a `Graph`.
Initially this seems quite limiting, but to allow custom logic for different types of graphs we have added graph hooks. These graph hooks allow custom code to be run before and after an operation chain is executed.

You can use hooks to do things like custom logging or special operation chain authorisation. To implement your own hook, just implement the `GraphHook` interface and register it with the graph when you build a `Graph` instance.
GraphHooks should be json serialisable and each hook should have a unit test that extends GraphHookTest.

There are some graph hooks which are added by default if they aren't already present in the configuration. The NamedViewResolver and NamedOperationResolver (providing that NamedOperations are supported by the store) are added at the start of the list of hooks.
The third hook added is the FunctionAuthoriser, which is added at the end of the list - again assuming no hook is not present in the configuration. This hook stops users from using potentially dangerous functions in their operation chain.
If you want to disable this hook, you should overwrite it by adding an empty FunctionAuthoriser to your list of hooks. For example:

```json
{
    "graphId": "example",
    "hooks": [
        {
            "class": "uk.gov.gchq.gaffer.graph.hook.FunctionAuthoriser"
        }
    ]
}
```
