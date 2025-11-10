# Bypass Aggregates for Context Multitenancy

## Summary

This branch adds support for `multitenancy: :bypass` option in aggregates when using context-based (schema-based) multitenancy in ash_postgres.

## What Was Done

### 1. Added Bypass Aggregates to Resources ✅

**File: `test/support/multitenancy/resources/user.ex`**

Added four bypass aggregates to the User resource:
- `posts_count_all_tenants` - COUNT aggregate that counts across all tenants
- `posts_count_current_tenant` - Normal COUNT aggregate for comparison
- `posts_list_all_tenants` - LIST aggregate across all tenants
- `has_posts_all_tenants` - EXISTS aggregate across all tenants

### 2. Added Comprehensive Tests ✅

**File: `test/multitenancy_test.exs`**

Added 4 test cases under "bypass aggregates for context multitenancy":
1. Counting across all tenants vs current tenant
2. LIST and EXISTS aggregates
3. Default values when no data exists
4. Using the `Ash.aggregate/3` API

### 3. Created Helper Module ✅

**File: `lib/aggregate/bypass.ex`**

Created a helper module with functions for:
- Detecting bypass aggregates
- Building UNION ALL queries
- Processing aggregates across multiple schemas

### 4. Documentation ✅

**Files:**
- `BYPASS_AGGREGATES_IMPLEMENTATION.md` - Detailed implementation guide
- `ash_sql_bypass_aggregates.patch` - Reference patch for ash_sql changes
- `BYPASS_AGGREGATES_README.md` - This file

## What Needs to Be Done

The actual implementation needs to be done in **ash_sql** because that's where aggregate queries are built.

### Required Changes in ash_sql

**File: `ash_sql/lib/aggregate.ex`**

The `add_subquery_aggregate_select` function needs to be modified to:

1. Detect aggregates with `multitenancy: :bypass` and context multitenancy strategy
2. Build UNION ALL queries across all tenant schemas
3. Combine results appropriately for each aggregate type (COUNT, LIST, EXISTS, etc.)

See `ash_sql_bypass_aggregates.patch` for a reference implementation.

### Why ash_sql and not ash_postgres?

- Aggregate query building happens in ash_sql
- ash_postgres delegates to ash_sql for aggregate processing
- The fix needs to be at the query building level
- ash_sql is shared by multiple data layers (postgres, mysql, sqlite)

## How to Test

1. Run the bypass aggregate tests:
   ```bash
   mix test test/multitenancy_test.exs:360
   ```

2. Expected results:
   - Before implementation: 4 failures
   - After implementation: 0 failures

## Example Usage

```elixir
defmodule MyApp.User do
  use Ash.Resource, ...

  aggregates do
    # Normal aggregate - counts only in current tenant schema
    count :posts_count, :posts do
      public?(true)
    end

    # Bypass aggregate - counts across ALL tenant schemas
    count :posts_count_all, :posts do
      multitenancy(:bypass)
      public?(true)
    end
  end
end

# Query with current tenant context
user = MyApp.User
  |> Ash.Query.load([:posts_count, :posts_count_all])
  |> Ash.Query.set_tenant("org_123")
  |> Ash.read_one!()

# posts_count: 5 (only in org_123 schema)
# posts_count_all: 50 (across all org_* schemas)
```

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Ash Framework                                               │
│ - Defines aggregates with multitenancy: :bypass option     │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ ash_postgres (DataLayer)                                    │
│ - Delegates aggregate queries to ash_sql                   │
│ - Provides repo.all_tenants() to get tenant list          │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ ash_sql (Query Builder) ⬅️ IMPLEMENTATION NEEDED HERE       │
│ - Detects bypass aggregates                                │
│ - Builds UNION ALL queries across tenant schemas          │
│ - Combines results per aggregate type                     │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL                                                  │
│ - Schema org_123.multitenant_posts (2 records)            │
│ - Schema org_456.multitenant_posts (3 records)            │
│ - Schema org_789.multitenant_posts (1 record)             │
│ └─> UNION ALL query returns: 6 total                      │
└─────────────────────────────────────────────────────────────┘
```

## Next Steps

1. **Apply changes to ash_sql**
   - Modify `ash_sql/lib/aggregate.ex`
   - Add bypass detection in `add_subquery_aggregate_select`
   - Implement UNION ALL query builder for each aggregate type

2. **Test the implementation**
   - Run `mix test test/multitenancy_test.exs:360`
   - Verify all 4 tests pass

3. **Performance optimization**
   - Cache tenant list
   - Add indexes on foreign keys
   - Consider adding configuration to limit tenant scope

## Files Changed in This Branch

```
Modified:
  test/multitenancy_test.exs                           (+164 lines)
  test/support/multitenancy/resources/user.ex          (+24 lines)

Created:
  lib/aggregate/bypass.ex                              (new file)
  BYPASS_AGGREGATES_IMPLEMENTATION.md                  (new file)
  BYPASS_AGGREGATES_README.md                          (new file)
  ash_sql_bypass_aggregates.patch                      (new file)
```

## References

- **Ash Multitenancy Docs**: https://hexdocs.pm/ash/multitenancy.html
- **PostgreSQL Schemas**: https://www.postgresql.org/docs/current/ddl-schemas.html
- **ash_postgres Multitenancy**: See `documentation/topics/advanced/schema-based-multitenancy.md`

## Contributors

- Initial implementation: Claude AI
- Based on requirements from: shahryarjb/ash_postgres

## Status

🟡 **Partial Implementation**
- ✅ Resource definitions
- ✅ Tests
- ✅ Documentation
- ⚠️ ash_sql query builder (needs implementation)
