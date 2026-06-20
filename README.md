# mcp-cluster-registry

The typed config schema for routing across N AtScale clusters from one gateway: parse a TOML or JSON file describing each cluster's endpoint, auth, capabilities, and priority, validate it, and query it.

## Why it exists

A single-cluster executor only needs to know one endpoint and one set of credentials. A federation gateway that fans out across several clusters needs more: which clusters exist, how to authenticate to each, what query backends each can serve, which one to prefer, and which models live where. Hand-rolling that as ad-hoc structs in the gateway means every downstream tool reinvents the same parsing and validation, and drifts.

This crate is the one place that shape is defined. It owns the `ClusterEntry` / `ClusterRegistry` types and nothing else — no network, no async, no credential values. You parse a config into a `ClusterRegistry`, validate it, and ask it questions. The gateway does the routing; this crate tells it what it's routing across.

## Install

It's a library crate, used as a git dependency:

```toml
# Cargo.toml
[dependencies]
mcp-cluster-registry = { git = "https://github.com/joeyen-atscale/mcp-cluster-registry" }
```

Requires Rust 1.70 or newer.

## Quickstart

Write a `clusters.toml`:

```toml
[[clusters]]
name = "prod"
endpoint = "mcp-aws.atscaleinternal.com:15432"
supported_backends = ["sql"]
priority = 0
required = true
tags = ["prod", "snowflake"]

[clusters.auth]
type = "direct"
pg_user = "atscale_user"
pg_pass_env = "PROD_PG_PASS"

[[clusters]]
name = "staging"
endpoint = "mcp-staging.atscaleinternal.com:15432"
xmla_url = "http://mcp-staging.atscaleinternal.com:11111"
supported_backends = ["sql", "dax", "mdx"]
priority = 1
required = false
model_filter = ["tpcds_benchmark_model"]

[clusters.auth]
type = "oidc"
token_url = "https://mcp-staging.atscaleinternal.com/auth/realms/AtScale/protocol/openid-connect/token"
client_id = "atscale-mcp"
realm = "AtScale"
client_secret_env = "STAGING_OIDC_SECRET"
```

Then parse, validate, and query it:

```rust
use mcp_cluster_registry::ClusterRegistry;

let toml = std::fs::read_to_string("clusters.toml")?;
let registry = ClusterRegistry::from_toml(&toml)?;

// validate() collects every problem, not just the first.
registry.validate().expect("registry should be valid");

// Highest-priority required cluster (lowest priority value).
let primary = registry.primary_cluster().unwrap();
assert_eq!(primary.name, "prod");

// Does a cluster declare a given query backend?
assert!(registry.supports_backend("staging", "dax"));
assert!(!registry.supports_backend("prod", "dax"));

// Clusters that can serve a model (no filter = any model).
let routable = registry.clusters_for_model("tpcds_benchmark_model");
assert_eq!(routable.len(), 2);
```

## The model

Two backing types and a handful of methods.

A **`ClusterEntry`** is one cluster: a `name`, a PGWire `endpoint`, an optional `xmla_url` for MDX/DAX, an `auth` config, the `supported_backends` it serves (a subset of `sql`/`dax`/`mdx`), an optional `model_filter`, a routing `priority` (lower wins), a `required` flag, and free-form `tags`.

A **`ClusterRegistry`** is the list of those, plus the methods to work with it:

| Method | What it returns |
|---|---|
| `from_toml(s)` / `from_json(s)` | Parse a config into a `ClusterRegistry`. |
| `to_json(&self)` | Serialize back to pretty JSON. |
| `validate(&self)` | `Ok(())`, or `Err(Vec<RegistryError>)` with *every* problem found. |
| `get(name)` | Look up one cluster by name. |
| `by_priority(&self)` | All clusters sorted ascending by `priority`. |
| `supports_backend(cluster, backend)` | Whether that cluster declares that backend. |
| `clusters_for_model(model_name)` | Clusters whose filter admits the model (or has no filter). |
| `primary_cluster(&self)` | Highest-priority `required` cluster, if any. |

Three properties the design holds to:

- **No credentials are ever stored.** `AuthConfig` holds only the env-var *names* (e.g. `pg_pass_env = "PROD_PG_PASS"`), never the values, and the serializer never calls `std::env::var()`. A test (`ac7_no_secret_leak`) enforces it: serialized output contains the variable names and nothing read from the environment.
- **TOML for people, JSON for machines.** You author in TOML; tools that pass the config around use JSON. `from_toml`, `from_json`, and `to_json` all produce or consume the same in-memory type, and the round-trip is tested.
- **Pure and in-memory.** Parsing and validation touch no network and no async runtime.

## Where it fits

It's the shared schema for the AtScale federation tooling — the crate every downstream federation tool depends on so they agree on what a cluster config looks like. It defines the contract; it does no routing or execution itself.

## Status

Early. The schema, parsing, validation, and query helpers are complete and covered by tests (acceptance criteria AC1–AC7). Validation currently checks for an empty cluster list, duplicate names, and missing endpoints; it does not yet cross-check, for example, that an MDX/DAX-capable cluster also declares an `xmla_url`. Version 0.1.0.

## License

MIT OR Apache-2.0
