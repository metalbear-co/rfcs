- Feature Name: `unified_branch_database_crd`
- Start Date: 2026-02-26
- Last Updated: 2026-03-02

## Summary

Replace the three database-specific Custom Resource Definitions (`PgBranchDatabase`, `MysqlBranchDatabase`, `MongodbBranchDatabase`) with a single unified `BranchDatabase` CRD. The new CRD uses a `dialect` field to pick the database and a `dialectOptions` field for db specific configuration, removing the code duplication across the CRD definitions, operator controllers, CLI, multi-cluster sync, and Helm chart.

## Motivation

The current database branching implementation defines a separate CRD for each supported database engine. While each CRD shares most of its structure (connection source, target, TTL, copy config, status), the database split forces duplication on a lot of places:

The three CRD specs are nearly identical - they are different only in the version field name (`postgresVersion` / `mysqlVersion` / `mongodbVersion`), the copy config entity name (`tables` vs `collections`), available copy modes, and IAM auth (Pg-only for now). Everything else (connection source, target, TTL, status, copy filters) is shared.

Adding a new database type requires touching many files with boilerplate, and bug fixes to shared logic must be replicated across all three paths.

## Guide-level explanation

### The unified `BranchDatabase` CRD

Instead of creating a `PgBranchDatabase`, `MysqlBranchDatabase`, or `MongodbBranchDatabase`, the CLI and operator now work with a single `BranchDatabase` resource. The `dialect` field tells the operator which database it is for:

```yaml
apiVersion: dbs.mirrord.metalbear.co/v1alpha1
kind: BranchDatabase
metadata:
  generateName: my-app-pg-branch-
spec:
  id: "abc123"
  dialect: postgres
  connectionSource:
    url:
      env:
        variable: DATABASE_URL
  databaseName: my_db
  target:
    deployment:
      name: my-app
  ttlSecs: 3600
  version: "16"
  copy:
    mode: schema
    items:
      users:
        filter: "username = 'alice'"
  dialectOptions:
    iamAuth:
      type: aws_rds
      region:
        env:
          variable: AWS_REGION
```

### Key changes from the user's perspective

- **No behavioral change**: database branching works exactly as before. The mirrord config file format is unchanged. Users do not need to modify their mirrord configuration.
- **Helm chart**: per-database flags are kept (`operator.pgBranching`, `operator.mysqlBranching`, `operator.mongodbBranching`, `operator.mssqlBranching`). All of them point to the same unified `BranchDatabase` CRD and controller, but the flags control which database engines are enabled. A single CRD definition is registered when any of the flags is enabled.

## Reference-level explanation

### Unified CRD spec

The three database CRD specs are replaced by a single `BranchDatabase` kind. The key new fields are:

- **`dialect`**: an enum with values `postgres`, `mysql`, `mongodb`, `mssql`. Tells the operator which database engine to provision.
- **`version`**: replaces the database version fields (`postgresVersion`, `mysqlVersion`, `mongodbVersion`) with a single optional string.
- **`dialectOptions`**: an optional section for db-specific configuration. Currently only contains `iamAuth` (used by PostgreSQL with AWS RDS / GCP Cloud SQL). As new dialect-specific features are needed, they are added here.

### Unified copy config

The database copy configs are merged. The `tables` field (Pg/MySQL/MSSQL) and `collections` field (MongoDB) become a single `items` field in the CRD - a map of entity name to an optional filter string. In the mirrord client config, each database type keeps its own field name (`tables` for SQL databases, `collections` for MongoDB) for clarity; the conversion to the unified `items` field happens when creating the CRD.

The copy mode is unified to `Empty`, `Schema`, `All`:

- **`empty`**: Creates an empty database with no schema or data. This is the default when the `copy` attribute is not specified.
- **`schema`**: Copies only the table structures (schemas) from the source database, without any data. If `items` (tables/collections) are specified, data is selectively copied for those items with their filters applied.
- **`all`**: Copies everything from the source database - both schema and all data. No filtering is supported in this mode.

Not all modes are valid for all engines. For example, MongoDB does not support `Schema` mode.

### Shared types (unchanged)

The connection source types, status/phase types, session info, and IAM auth config all remain unchanged from the current design.

### Approach A: (keep old CRDs temporarily)

Run old and new CRDs side-by-side for some time, giving users time to upgrade CLI and operator.

**Introduction release**:

1. Helm chart registers the new `BranchDatabase` CRD alongside the three old ones.
2. Operator starts controllers for both old (`PgBranchDatabase`, `MysqlBranchDatabase`, `MongodbBranchDatabase`) and new (`BranchDatabase`) CRD types.
3. New CLI versions create only the new `BranchDatabase` CRD.
4. Old CRDs still work - users with older CLI versions continue working.
5. Multi-cluster sync controller watches all CRD kinds.

**Later removal release**:

1. Old CRD definitions are removed from the Helm chart.
2. Old controllers and sync watchers are removed from the operator.

**Pros**:

- Zero downtime for users during transition.

**Cons**:

- Operator must maintain both old and new controller code for a few releases.
- Multi-cluster sync complexity increases during the overlap period.
- More testing (both ways must work).

### Approach B: (remove old CRDs immediately)

Single release replaces old CRDs with the new one. No overlap period.

### Switchover release

1. Helm chart removes the three old CRD definitions and adds the single `BranchDatabase` CRD.
2. Operator ships only the new `BranchDatabase` controller.
3. CLI creates only the new `BranchDatabase` CRD.
4. When the Helm upgrade runs, Kubernetes deletes the old CRD definitions, which also deletes all existing old CRD instances and their owned pods.

**Active sessions at upgrade time**: any branch database pods that exist at upgrade time are deleted because removing a CRD definition also removes all its instances. Users with active mirrord sessions using database branches will see their sessions fail. This is acceptable because:

- Users can restart their mirrord session, which creates a new branch using the new CRD.
- The upgrade window can be communicated in release notes.

**Pros**:

- No extra complexity - clean, simple upgrade.
- Operator codebase immediately benefits from less duplication (no legacy code).

**Cons**:

- Breaking change for any active sessions at upgrade time.

## Rationale and alternatives

### Why this design

A single CRD with a `dialect` field is the natural fit for resources that share most of their schema. It matches how the operator already works internally: the controller is generic and the database-specific logic lives in separate initializer implementations. The unified CRD makes the Kubernetes API match the internal design.

### Impact of not doing this

The existing architecture works. The cost is ongoing maintenance: every new feature or bug fix in the branching flow must be applied to x number of places depending on how many database types we have, X CRD schemas, X controllers, and X sync controllers.

## Unresolved questions

- **Copy mode validation**: should `Schema` mode be added to MongoDB as a no-op (always valid, just does nothing extra), or should it be rejected at runtime?
- **Naming**: `dialect` vs `engine` vs `type` for the database kind field. `dialect` is used in this RFC but `engine` may be more intuitive. and if renamed the options should probably also follow. For example, if we go with `engine` - `engineOptions`, `dialect` - `dialectOptions`, `type` - `typeOptions` etc..
- **Helm flag naming**: `operator.dbBranching` vs keeping separate database flags that all point to the same CRD.

## Future possibilities

- **New database engines**: with the unified CRD, adding support for a new engine (CockroachDB etc.) requires only adding a new dialect variant, implementing the database-specific initializer, and adding the init container binary. No CRD schema changes, no Helm chart changes, no new controllers.
- **`dialectOptions` extensibility**: the flat approach makes it easy to add new dialect-specific fields without changing the API version. For example, MongoDB-specific replica set configuration or MySQL-specific character set defaults could be added as optional fields.
