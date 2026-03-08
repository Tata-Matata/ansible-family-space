# What a Consul server is allowed to do

# Each policy applies to all objects whose name starts with this prefix

# controls Node metadata, registrations, health association
# empty strings = all nodes
# node = logical member of the Consul catalog (consul-1, vault-1, worker-2)
# A node exists because a Consul agent runs on it (so for now it is only consul-1)
# A node record contains node name, address, which services/checks belong to it
# node write = registering a node, updating node metadata, associating checks with the node, participate in cluster membership
# with the current setup of only one consul, the consul server can ensure it exists in the catalogue, keep its node record fresh, tie Raft/session state to itself
# Without it the server process starts but cannot fully register itself

# API examples: PUT /v1/catalog/register, PUT /v1/catalog/deregister

node_prefix "" {
  policy = "write"
}

# controls Service registrations (s. below), service health checks
# currently, with one Consul, we only have one consul service
# the write policy allows the Consul server to declare its own internal services (this one consul service for now) in the catalogue, 
# attach health checks to those services (internal RPC / HTTPS checks), update service health (leader and Raft-related checks)
# without it internal services fail to register, health checks misbehave, the catalog becomes inconsistent
# Later: K8s workloads use Consul for discovery
# Pods register services in Consul, Consul becomes a discovery layer

service_prefix "" {
  policy = "write"
}

# controls local agent operations (checks, services)
# apply only to what this Consul agent is allowed to do as a process.
# do not grant control over other agents.
# A Consul agent is a local process with a local HTTP API. 
# When an agent registers a local service, registers a local health check, updates the status of that check
# it is calling its own agent API
agent_prefix "" {
  policy = "write"
}

# controls Sessions & locks (leader locks, Vault HA coordination, s. explanation below)
session_prefix "" {
  policy = "write"
}

# controls KV store 
key_prefix "" {
  policy = "write"
}

# controls ACL system itself (policies, tokens)
# does not allow creating policies, tokens, ACLs
acl = "read"


#### NODE, AGENT, SERVICE 
# Consul has 3 levels of identity: agent --> node --> services
# Agent = the Consul binary running on a machine (like CI/CD agent)
# The policy for Agent: “What is this Consul process allowed to do locally?”
# Node — the identity in the cluster. “The catalog identity that an agent represents.”
# This node identity owns sessions, owns locks, groups checks and services, is what the cluster reasons about
# service = “Something this node claims to provide.”
# Services live on nodes, are optional, are what discovery usually cares about

# one agent can manage multiple services
# services can move between nodes
# nodes can exist with no services


# Registering a service is performed in 2 steps
# agent_prefix controls the act of registering it locally
# service_prefix controls the right for that service to exist in the catalog
# Both are required for the same service to appear and function correctly.

# STEP 1
# The Consul agent process decides: “I provide a service called consul and here are its health checks.”
# This is a local operation, handled by the agent API: PUT /v1/agent/service/register
# Without this the agent is not allowed to even declare the service locally, registration fails immediately

# STEP 2
# Catalog-level action (global): Once the agent accepts the service locally, Consul then says:
# “This service named consul now exists on node consul-1 in the cluster catalog.”
# This is a cluster-wide fact, stored in the catalog.
# Without it, the service never truly exists for the cluster


#### SESSION

# A lease that ties ownership of something (usually a lock) to a client, 
# and automatically expires if that client disappears.
# the write policy allows the Consul server to create, renew, destroy sessions,associate sessions with internal locks
# These map to API calls like: PUT /v1/session/create, PUT /v1/session/renew, PUT /v1/session/destroy
# Consul server processes (control planes) coordinate Consul cluster state and safety.
# consul-server process calls internal coordination logic, which uses the same session machinery exposed via the API
# so Consul server needs to initiate sessions to  call its own API


#### KEY 
# A key is an entry in Consul’s KV store (vault/core/, vault/logical/, locks/service-x, config/app1)
# This is not service discovery. This is state storage.
# Consul servers are the authority for the KV store. 
# Although Vault writes KV via its own token, Consul Servers “just replicate” them via Raft,
# in practice, some internal paths touch KV state for upgrades, snapshots, disaster recovery
# HashiCorp’s own examples always include it for servers.
