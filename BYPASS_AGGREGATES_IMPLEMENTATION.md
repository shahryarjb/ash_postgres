# Bypass Aggregates for Context Multitenancy - Implementation Guide

## Overview

This document describes the implementation of `multitenancy: :bypass` for aggregates when using context-based multitenancy in ash_postgres.

## Problem Statement

When using schema-based multitenancy (`:context` strategy), aggregates normally query only the current tenant's schema. For example, if the current tenant is `org_123`, an aggregate will only query tables in the `org_123` schema.

However, sometimes you need to aggregate data across **ALL** tenant schemas. This is what the `multitenancy: :bypass` option enables.

## Current Status

### What Works ✅
- Attribute-based multitenancy bypass (implemented in Ash framework)
- Bypass aggregates are defined in resources (see `test/support/multitenancy/resources/user.ex`)
- Comprehensive tests are written (see `test/multitenancy_test.exs`)

### What Needs Implementation ⚠️
The actual query building logic in `ash_sql` needs to be modified to:
1. Detect aggregates with `multitenancy: :bypass`
2. Build UNION ALL queries across all tenant schemas
3. Combine and return the aggregated results

## Implementation Location

The fix needs to be implemented in **ash_sql** because that's where aggregate queries are built. Specifically:

### File: `ash_sql/lib/aggregate.ex`

The `add_subquery_aggregate_select` function (around line 2335) needs to be modified to detect bypass aggregates and build UNION ALL queries.

### Current Behavior
```elixir
# In ash_sql/lib/aggregate.ex, around line 2335
field =
  case kind do
    :count ->
      cond do
        !aggregate.field ->
          Ecto.Query.dynamic([row], count())
        # ... more conditions
      end
  end
```

### Required Behavior
```elixir
field =
  case kind do
    :count ->
      # Check for bypass aggregates with context multitenancy
      if Map.get(aggregate, :multitenancy) == :bypass &&
         Ash.Resource.Info.multitenancy_strategy(resource) == :context do
        # Build UNION ALL query across all tenant schemas
        build_bypass_count_aggregate(aggregate, resource, repo)
      else
        # Normal count aggregate (current behavior)
        cond do
          !aggregate.field ->
            Ecto.Query.dynamic([row], count())
          # ... existing logic
        end
      end
  end
```

## Implementation Steps

### Step 1: Add Helper Functions (in ash_sql/lib/aggregate.ex)

```elixir
defp has_bypass_multitenancy?(aggregates) do
  Enum.any?(aggregates, fn agg ->
    Map.get(agg, :multitenancy) == :bypass
  end)
end

defp get_all_tenants(resource, repo) do
  if function_exported?(repo, :all_tenants, 0) do
    repo.all_tenants()
  else
    []
  end
end

defp build_bypass_aggregate(aggregate, resource, repo) do
  all_tenants = get_all_tenants(resource, repo)

  if all_tenants == [] do
    # No tenants, return 0 or default
    Ecto.Query.dynamic([row], 0)
  else
    # Build UNION ALL query across all tenant schemas
    build_union_all_aggregate(aggregate, resource, all_tenants)
  end
end
```

### Step 2: Build UNION ALL Query

```elixir
defp build_union_all_aggregate(aggregate, resource, all_tenants) do
  # This is the complex part that needs careful implementation
  #
  # Pseudocode:
  # 1. For each tenant schema:
  #    - Build a subquery that counts/aggregates in that schema
  # 2. Combine all subqueries with UNION ALL
  # 3. Wrap in a final aggregate (SUM of counts, etc.)

  # Example for COUNT:
  # SELECT SUM(cnt) FROM (
  #   SELECT COUNT(*) as cnt FROM org_123.posts WHERE ...
  #   UNION ALL
  #   SELECT COUNT(*) as cnt FROM org_456.posts WHERE ...
  # ) AS all_counts

  # This requires building an Ecto.Query.dynamic that represents
  # a subquery with UNION ALL across all schemas
end
```

### Step 3: Modify Each Aggregate Type

For each aggregate type (COUNT, LIST, EXISTS, SUM, MAX, MIN, AVG, FIRST), add bypass detection:

```elixir
:count ->
  if has_bypass?(aggregate, resource) do
    build_bypass_count(aggregate, resource, repo)
  else
    # existing logic
  end

:list ->
  if has_bypass?(aggregate, resource) do
    build_bypass_list(aggregate, resource, repo)
  else
    # existing logic
  end

# ... and so on for each aggregate type
```

## Alternative: Implement in ash_postgres

If modifying ash_sql is not feasible, the implementation can be done in ash_postgres by:

### 1. Override run_aggregate_query_with_lateral_join

In `ash_postgres/lib/data_layer.ex`, intercept bypass aggregates:

```elixir
def run_aggregate_query_with_lateral_join(
      query,
      aggregates,
      root_data,
      destination_resource,
      path
    ) do
  # Check if any aggregates have bypass
  {bypass_aggs, normal_aggs} =
    Enum.split_with(aggregates, fn agg ->
      Map.get(agg, :multitenancy) == :bypass &&
      Ash.Resource.Info.multitenancy_strategy(destination_resource) == :context
    end)

  if bypass_aggs != [] do
    # Handle bypass aggregates separately
    bypass_results = process_bypass_aggregates(
      query,
      bypass_aggs,
      root_data,
      destination_resource,
      path
    )

    # Handle normal aggregates with existing logic
    normal_results = call_original_lateral_join(
      query,
      normal_aggs,
      root_data,
      destination_resource,
      path
    )

    # Merge results
    merge_aggregate_results(bypass_results, normal_results)
  else
    # No bypass aggregates, use existing logic
    # ... existing implementation
  end
end
```

### 2. Implement process_bypass_aggregates

```elixir
defp process_bypass_aggregates(
       query,
       bypass_aggregates,
       root_data,
       destination_resource,
       path
     ) do
  repo = AshPostgres.DataLayer.Info.repo(destination_resource)
  all_tenants = repo.all_tenants()

  # For each bypass aggregate, query across all tenant schemas
  Enum.map(bypass_aggregates, fn agg ->
    results =
      Enum.map(all_tenants, fn tenant ->
        # Build query for this tenant
        tenant_query = build_tenant_query(query, agg, tenant, destination_resource)

        # Execute query
        repo.all(tenant_query)
      end)
      |> List.flatten()

    # Aggregate the results based on aggregate type
    combine_tenant_results(agg, results)
  end)
end
```

## Testing

Tests have been added in `test/multitenancy_test.exs` under the `"bypass aggregates for context multitenancy"` describe block.

To run the tests:

```bash
mix test test/multitenancy_test.exs:360
```

## Expected Test Results

Before implementation:
```
5 tests, 5 failures
```

After implementation:
```
5 tests, 0 failures
```

## Files Modified

1. `test/support/multitenancy/resources/user.ex` - Added bypass aggregate definitions
2. `test/multitenancy_test.exs` - Added comprehensive tests
3. `lib/aggregate/bypass.ex` - Helper module for bypass logic (created)
4. `BYPASS_AGGREGATES_IMPLEMENTATION.md` - This file

## Files That Need Modification (in ash_sql or ash_postgres)

1. `ash_sql/lib/aggregate.ex` - Add bypass detection and UNION ALL query building
   OR
2. `ash_postgres/lib/data_layer.ex` - Override aggregate query functions to handle bypass

## Next Steps

1. Decide whether to implement in ash_sql or ash_postgres
2. Implement the UNION ALL query builder
3. Handle all aggregate types (COUNT, LIST, EXISTS, SUM, MAX, MIN, AVG, FIRST)
4. Run tests to verify
5. Optimize performance (caching tenant list, etc.)

## Performance Considerations

- UNION ALL across many tenant schemas can be expensive
- Consider adding an index on the relationship foreign keys
- Consider caching the tenant list
- Consider adding a configuration option to limit which tenants are queried
