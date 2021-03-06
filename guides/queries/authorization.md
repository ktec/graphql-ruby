---
title: Queries — Authorization
---

GraphQL offers a few ways to ensure that clients access data according to their permissions.

- __Query analyzers__ can assert that the query is valid before running it.
- __Resolve wrappers__ can assert that returned objects are permitted to a given user.

## Query Analyzers

A query analyzer visits each field in the query _before_ the query is executed. It can accumulate data during the visits, then return a value. If the returned value is a {{ "GraphQL::AnalysisError" | api_doc }} (or an array of those errors), the query won't be executed and the error will be returned to the user. You can use this feature to assert that queries are permitted _before_ running them!

Query analyzers reuse concepts from `Array#reduce`, so let's briefly revisit how that method works:

```ruby
items = [1, 2, 3, 4, 5]
initial_value = 0
reduce_result = items.reduce(0) { |memo, item| memo + item }
final_value = "Sum: #{reduce_result}"
puts final_value
# Sum: 15
```

- `reduce` accepts an _initial value_ and a _callback_ (as a block)
- The callback receives the reduce _state_ (`memo`) and each item of the array (`item`)
- For each call to the callback, the return value is the new state and it will be provided to the _next_ call to the callback
- When each item has been visited, the last value of the callback state (the last `memo` value) is returned
- Then, you can use the reduced value in your application

A query analyzer has the same basic parts. Here's the scaffold for an analyzer:

```ruby
class MyQueryAnalyzer
  # Called before the visit.
  # Returns the initial value for `memo`
  def initial_value(query)
  end

  # This is like the `reduce` callback.
  # The return value is passed to the next call as `memo`
  def call(memo, visit_type, irep_node)
  end

  # Called when we're done the whole visit.
  # The return value may be a GraphQL::AnalysisError (or an array of them).
  # Or, you can use this hook to write to a log, etc
  def final_value(memo)
  end
end
```

- `#initial_value` is a chance to initialize the state for your analysis. For example, you can return a hash with keys for the query, schema, and any other values you want to store.
- `#call` is called for each node in the query. `memo` is the analyzer state. `visit_type` is either `:enter` or `:leave`. `irep_node` is the {{
  "GraphQL::InternalRepresentation::Node" | api_doc }} for the current field in the query. (It is like `item` in the `Array#reduce` callback.)
- `#final_value` is called _after_ the visit. It provides a chance to write to your log or return a {{ "GraphQL::AnalysisError" | api_doc }} to halt query execution.

Query analyzers are added to the schema with `query_analyzer`, for example:

```ruby
MySchema = GraphQL::Schema.define do
  query_analyzer MyQueryAnalyzer.new
end
```

GraphQL's `max_depth` and `max_complexity` are implemented with query analyzers, you can see those for reference:

- {{ "GraphQL::Analysis::QueryDepth" | api_doc }}
- {{ "GraphQL::Analysis::QueryComplexity" | api_doc }}

## Resolve Wrapper

Sometimes, you can only check permissions when you have the _actual_ object. Let's say you're exposing documents in your API:

```ruby
field :documents, types[DocumentType] do
  resolve ->(obj, args, ctx) {
    documents = obj.documents
    # sort, filter, etc
    # return the documents:
    documents
  }
end
```

You can "wrap" this resolve function to assert that the documents are ok for the current user:

```ruby
# Take a resolve function and call it.
# Then, check that the result includes permitted records _only_.
# @return [Proc] a new resolve function that checks the return values
def assert_allowed_documents(resolve_func)
  ->(obj, args, ctx) {
    documents = resolve_func.call(obj, args, ctx)
    current_user = ctx[:current_user]

    if documents.all? { |d| current_user.can_view?(d) }
      documents
    else
      nil
    end
  }
end

# ...

field :documents, types[DocumentType] do
  # wrap the resolve function with your assertion
  resolve assert_allowed_documents(->(obj, args, ctx) {
    # ...
  })
end
```

This way, you can "catch" the returned value before giving it to a client.

This approach can be further parameterized by implementing it as a class, for example:

```ruby
# Assert that the current user has `permission` on the return value of `block`
class PermissionAssertion
  # Get a permission level and the "inner" resolve function
  def initialize(permission, resolve_func)
    @permission = permission
    @resolve_func = resolve_func
  end

  # GraphQL will call this, so delegate to the "inner" resolve function
  # and check the return value
  def call(obj, args, ctx)
    value = @resolve_func.call(obj, args, ctx)
    current_user = ctx[:current_user]
    if current_user.can?(@permission, value)
      value
    else
      nil
    end
  end
end

# ...

# Apply this class to the resolve function:
field :documents, types[DocumentType] do
  resolve PermissionAssertion.new(:view, ->(obj, args, ctx) {
    # ...
  })
end
```
