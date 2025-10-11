---
layout: post
title: "Solving Elixir-Neo4j Connectivity: From gRPC to Rust NIFs"
date: 2025-10-11
categories: elixir rust neo4j nif grpc
excerpt: "When bolt_sips had SSL and dependency conflicts, I explored two paths to reliable Neo4j connectivity in Elixir."
---

# Solving Elixir-Neo4j Connectivity: From gRPC to Rust NIFs

Building Grimoire, my AI-driven interactive platform, I needed reliable Neo4j connectivity from Elixir. The existing Elixir library had blocking issues. Here's how I solved it through iteration.

## The Problem: bolt_sips Wasn't Working

I needed to connect my Elixir/Phoenix application to Neo4j Aura (Neo4j's cloud offering). The natural choice was [bolt_sips](https://github.com/florinpatrascu/bolt_sips), the established Elixir driver for Neo4j's Bolt protocol.

**Two problems stopped me cold:**

1. **SSL Invalid Key Error**: When connecting to Neo4j Aura, bolt_sips threw SSL certificate validation errors. Aura requires secure connections, and the SSL handshake was failing.

2. **Dependency Conflict**: bolt_sips conflicted with the official Firestore Elixir driver. Since Grimoire uses both Neo4j (for graph relationships) and Firestore (for document storage), this was a non-starter.

I needed a solution that would give me reliable Aura connectivity without abandoning either database.

## Approach 1: gRPC to a Rust Service

My first instinct was to isolate the Neo4j connectivity problem into a separate service using the mature Rust Neo4j driver.

### The Design

```
[Elixir/Phoenix App] <--gRPC--> [Rust Service] <--Bolt--> [Neo4j Aura]
```

**Why gRPC made sense initially:**
- Language-agnostic interface
- Could use the official [neo4rs](https://github.com/neo4j-labs/neo4rs) Rust driver
- Service isolation - Neo4j problems wouldn't crash my Phoenix app
- No dependency conflicts in the Elixir app

### The Implementation Problem

I got a basic gRPC service working in Rust with neo4rs. The Rust side handled connections beautifully - no SSL issues, solid performance.

**But the interface felt wrong.**

To use gRPC effectively, I'd need to define the service operations upfront:

```protobuf
service Neo4jService {
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc FindRelationships(RelationshipQuery) returns (Relationships);
  rpc ExecuteCustomQuery(CustomQueryRequest) returns (QueryResult);
  // ... dozens more domain-specific methods
}
```

This meant either:
- **Option A**: Create specific RPC methods for every query pattern (maintenance nightmare)
- **Option B**: Pass Cypher queries as strings through gRPC (losing type safety and adding network overhead)

### Why I Moved On

What I really wanted was to **write Cypher queries directly in my Elixir code**:

```elixir
# What I wanted to write
query = """
  MATCH (u:User)-[:INTERESTED_IN]->(t:Topic)
  WHERE u.id = $user_id
  RETURN t.name, t.category
"""

Neo4j.execute(query, %{user_id: user_id})
```

The gRPC approach forced me to either:
- Abstract away Cypher (losing query flexibility)
- Add a generic query endpoint (adding network latency for every query)
- Maintain parallel query definitions in proto files and Elixir

None of these felt right. I wanted Neo4j connectivity to feel native in Elixir, not like a remote service call.

## Approach 2: Rust NIF Wrapper

What if I could bring the Rust Neo4j driver directly into the BEAM VM?

### Native Implemented Functions (NIFs)

NIFs let you call native code (C, Rust, etc.) from Elixir. The Rust ecosystem has excellent tooling for this via [Rustler](https://github.com/rusterlium/rustler).

**The key insight:** Wrap the official neo4rs Rust driver in a NIF, giving me:
- Reliable Aura connectivity (no SSL issues)
- No dependency conflicts (separate native library)
- Native-speed database operations
- Elixir-native API (write Cypher in Elixir code)
- Single deployment unit (no microservice complexity)

### Implementation Strategy

```rust
use rustler::{Env, Term, NifResult, ResourceArc};
use neo4rs::{Graph, query, ConfigBuilder};

// Wrap the Neo4j Graph in a resource
struct GraphResource {
    graph: Graph,
}

#[rustler::nif]
fn connect(uri: String, user: String, password: String) -> NifResult<ResourceArc<GraphResource>> {
    let config = ConfigBuilder::default()
        .uri(&uri)
        .user(&user)
        .password(&password)
        .build()?;
    
    let graph = Graph::connect(config).await?;
    
    Ok(ResourceArc::new(GraphResource { graph }))
}

#[rustler::nif(schedule = "DirtyCpu")]
fn execute_query(
    graph: ResourceArc<GraphResource>,
    query_str: String,
    params: Term
) -> NifResult<Vec<Term>> {
    // Convert Elixir params to Neo4j query parameters
    // Execute query on the graph
    // Convert results back to Elixir terms
}
```

**Key design decisions:**

1. **Dirty Schedulers**: Database queries can take significant time. Using `schedule = "DirtyCpu"` ensures long-running queries don't block BEAM schedulers.

2. **Resource Management**: The `GraphResource` wrapper ensures proper connection lifecycle. When the Elixir process holding the resource dies, Rust cleanup runs automatically.

3. **Cypher in Elixir**: The NIF accepts Cypher query strings and parameters from Elixir, executing them directly through neo4rs.

### The Elixir API

On the Elixir side, I built a clean API that feels native:

```elixir
defmodule Grimoire.Neo4j do
  @moduledoc """
  Neo4j graph database interface using Rust NIF.
  """

  # Connection management
  def connect(uri, user, password) do
    Neo4jNif.connect(uri, user, password)
  end

  # Execute Cypher queries
  def execute(graph, query, params \\ %{}) do
    Neo4jNif.execute_query(graph, query, params)
  end

  # Helper for common patterns
  def find_relationships(graph, user_id) do
    query = """
      MATCH (u:User)-[r]->(n)
      WHERE u.id = $user_id
      RETURN type(r) as relationship, n
    """
    
    execute(graph, query, %{user_id: user_id})
  end
end
```

Now I can write Cypher directly in my Elixir code, with all the flexibility of the query language, but backed by the reliable Rust driver.

### Error Handling

Rust errors convert cleanly to Elixir exceptions:

```rust
// In the NIF
match graph.execute(query).await {
    Ok(result) => Ok(convert_to_elixir(result)),
    Err(e) => Err(rustler::Error::Term(Box::new(format!("Neo4j error: {}", e))))
}
```

```elixir
# In Elixir
try do
  Neo4j.execute(graph, query, params)
rescue
  e in RuntimeError -> 
    Logger.error("Query failed: #{Exception.message(e)}")
    {:error, :query_failed}
end
```

## The Result

This solution gives me:

✅ **Reliable Aura connectivity** - neo4rs handles SSL perfectly  
✅ **No dependency conflicts** - NIF is isolated from other Elixir dependencies  
✅ **Native performance** - Direct Rust execution, no network overhead  
✅ **Cypher in Elixir** - Write queries where I need them  
✅ **Simple deployment** - Single release, no microservices  
✅ **BEAM supervision** - Connection failures handled by supervision trees

## Lessons Learned

### 1. Iteration Reveals the Right Solution

I didn't know NIFs were the answer until gRPC showed me what I *actually* wanted: Cypher queries in Elixir code. The gRPC attempt wasn't wasted - it clarified my requirements.

### 2. Developer Experience Matters

The gRPC approach would have worked functionally, but the developer experience was poor:
- Define operations in proto files
- Generate code
- Marshal data across the network
- Debug across service boundaries

The NIF approach feels like Neo4j is just another Elixir library, which is exactly what I wanted.

### 3. Embrace Polyglot When Needed

Elixir excels at concurrency and fault tolerance. Rust excels at performance and systems integration. NIFs let me use both where they shine best.

### 4. Operational Simplicity Wins

**gRPC approach complexity:**
- Deploy and monitor a separate service
- Handle network failures between services
- Coordinate version updates
- Debug distributed traces

**NIF approach complexity:**
- Compile Rust code during Elixir build
- Ensure NIF safety (dirty schedulers, proper resource cleanup)
- Handle Rust-to-Elixir type conversions

The NIF complexity is front-loaded (implementation time) but doesn't add ongoing operational overhead.

## When to Use This Pattern

A Rust NIF wrapper makes sense when:

✅ The BEAM ecosystem lacks a mature library  
✅ A solid Rust library exists (officially supported is even better)  
✅ You want native-feeling APIs in Elixir  
✅ You can handle NIF safety concerns properly  
✅ Operational simplicity is worth implementation complexity

It's probably overkill when:

❌ An adequate pure-Elixir library exists  
❌ HTTP/REST API would suffice  
❌ Your team isn't comfortable with Rust  
❌ The added complexity doesn't justify the benefits

## Next Steps

I'm considering open-sourcing the Neo4j NIF wrapper. It could help others facing similar connectivity issues with bolt_sips, especially for Aura deployments.

If you're interested or have questions about the implementation, feel free to reach out.

---

*Building Grimoire has been an exercise in pragmatic technology choices. Sometimes the right solution means reaching outside your primary stack - but keeping the complexity where it belongs.*