# Kiali Multi-Mesh Architecture for External Control Planes

This document describes architectural changes enabling Kiali to observe **multiple independent meshes** on a shared infrastructure cluster, where each mesh has its own external Istio control plane managing a different overlay (remote) cluster.

## Problem Statement

Kiali was designed for a **single mesh** topology: one control plane managing one or more clusters. In the **applink / multi-mesh** model, a single infrastructure ("underlay") cluster hosts multiple istiod instances in separate namespaces, each acting as an external control plane for a different overlay cluster. A single Kiali instance must observe all of these meshes together.

## Deployment Topology

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Underlay (Infrastructure) Cluster                │
│                                                                     │
│  ┌──────────────────────┐                                           │
│  │  Kiali Namespace      │  ◄── Single Kiali instance               │
│  │  (management NS)      │      observes all meshes below           │
│  └──────────┬───────────┘                                           │
│             │                                                       │
│  ┌──────────▼───────────┐    ┌──────────────────────┐               │
│  │  MCP Namespace A      │    │  MCP Namespace B      │              │
│  │  ┌──────────────┐    │    │  ┌──────────────┐    │              │
│  │  │   istiod      │    │    │  │   istiod      │    │              │
│  │  │ CLUSTER_ID=   │    │    │  │ CLUSTER_ID=   │    │              │
│  │  │ "mcp-ns-a-id" │    │    │  │ "mcp-ns-b-id" │    │              │
│  │  └──────┬───────┘    │    │  └──────┬───────┘    │              │
│  └─────────┼────────────┘    └─────────┼────────────┘              │
│            │                           │                            │
│  ┌─────────┼───────────────────────────┼──────────┐                │
│  │         │     Prometheus            │          │                │
│  │         │  (one per Kiali group)    │          │                │
│  │         │  Scrapes both overlays    │          │                │
│  └─────────┼───────────────────────────┼──────────┘                │
└────────────┼───────────────────────────┼───────────────────────────┘
             │ manages                   │ manages
             ▼                           ▼
┌────────────────────────┐   ┌────────────────────────┐
│   Overlay Cluster A    │   │   Overlay Cluster B    │
│   (e.g. "customer-1")  │   │   (e.g. "customer-2")  │
│                        │   │                        │
│  Workloads, services   │   │  Workloads, services   │
│  ztunnel, waypoints    │   │  ztunnel, waypoints    │
└────────────────────────┘   └────────────────────────┘
```

### Key characteristics

- **One Kiali per group** of meshes, NOT one per mesh
- **Each MCP namespace** contains its own istiod (`EXTERNAL_ISTIOD=true`)
- **Each istiod manages exactly one overlay cluster** via remote config
- **One Prometheus** scrapes all overlay clusters in the group
- **istiod's `CLUSTER_ID`** (an opaque ID like a namespace hash) differs from the overlay cluster's name in Kiali

## Gaps Addressed

### Gap 1: Control Plane Discovery Scope

**Problem:** Kiali scans ALL `app=istiod` deployments cluster-wide. On a shared underlay, it discovers istiods belonging to *other* Kiali groups.

**Solution:** Added `control_plane_namespace_selector` config — a label selector that scopes istiod discovery to matching namespaces only. When unset, behavior is unchanged (scans all).

```
┌─ Underlay Cluster ──────────────────────────────────────────┐
│                                                              │
│  NS: mcp-a  (label: mesh-group=group-1)  ── istiod ✓       │
│  NS: mcp-b  (label: mesh-group=group-1)  ── istiod ✓       │
│  NS: mcp-c  (label: mesh-group=group-2)  ── istiod ✗ skip  │
│                                                              │
│  Kiali config:                                               │
│    control_plane_namespace_selector:                         │
│      matchLabels:                                            │
│        mesh-group: "group-1"                                 │
└──────────────────────────────────────────────────────────────┘
```

**Files:** `istio/discovery.go` — pre-filters namespaces before scanning for istiod deployments.

### Gap 3: Mesh-Aware Business Layer

**Problem:** Business layer services (e.g., `GetIstioConfigMap`) loop over ALL clusters without knowing which clusters belong to which mesh. Querying a namespace on a cluster where it doesn't exist causes errors.

**Solution:** Added mesh-awareness via `MeshDiscovery.GetClustersForMesh()` — resolves which clusters are managed by a given control plane. `IstioConfigCriteria` gains a `Cluster` field to scope config fetches to specific clusters.

```
Before (Gap 3):                       After (Gap 3):

GetIstioConfigMap(ns="default")       GetIstioConfigMap(ns="default",
  → loop ALL clusters                   cluster="customer-1")
  → query underlay ❌ (no "default")    → query only customer-1 ✓
  → query customer-1 ✓                  → skip unrelated clusters
  → query customer-2 ✗ (wrong mesh)
```

**Files:** `business/istio_config.go`, `business/layer.go`, `istio/discovery.go`

### Gap 5: Metric Cluster ID → Name Mapping

**Problem:** istiod sets `CLUSTER_ID` on data plane proxies (ztunnel/waypoints). Prometheus metrics report this as `source_cluster` / `destination_cluster`. But Kiali knows the cluster by a different name (from kubeconfig secrets). Graph nodes get wrong cluster IDs and navigation breaks.

```
Prometheus metric:  source_cluster="mcp-ns-a-id"     (istiod CLUSTER_ID)
Kiali cluster name:                 "customer-1"      (from kubeconfig secret)

Without mapping:  Graph shows "mcp-ns-a-id"  →  click  →  cluster not found ❌
With mapping:     Graph shows "customer-1"    →  click  →  navigates correctly ✓
```

**Solution:** Auto-discover the mapping from mesh topology. `MeshDiscovery.GetClusterNameMapping()` iterates control planes: when a control plane's `CLUSTER_ID` differs from its managed cluster's name, it records the translation. `HandleClusters()` in the graph telemetry layer applies this mapping to all Prometheus metric labels.

```
Discovery:
  istiod in mcp-a:  CLUSTER_ID = "mcp-ns-a-id"
                     manages     → "customer-1"
  → mapping: { "mcp-ns-a-id" → "customer-1" }

HandleClusters(sourceCluster="mcp-ns-a-id", mapping):
  → returns "customer-1"
```

**Files:** `istio/discovery.go`, `graph/telemetry/istio/util/util.go`, `graph/telemetry/istio/appender/*.go`, `graph/api/api.go`

### Gap 6: Home Cluster Identity

**Problem:** In the external control plane model, istiod's `CLUSTER_ID` is an opaque ID, not the Kiali cluster name. Discovery logic that matches `controlPlane.ID == cluster.Name` fails when they differ.

**Solution:** Resolved together with Gap 5. The `GetClusterNameMapping()` function handles the identity mismatch by using annotation-based matching (`topology.istio.io/controlPlaneClusters`) and building the translation table. Kiali's home cluster identity (the underlay) is kept separate from control plane identities.

## Data Flow

```
                    ┌──────────────┐
                    │  Prometheus   │
                    │  (per group)  │
                    └──────┬───────┘
                           │ source_cluster="mcp-ns-a-id"
                           │ destination_cluster="mcp-ns-b-id"
                           ▼
                    ┌──────────────┐
                    │ HandleClusters│ ◄── clusterNameMapping:
                    │  (util.go)   │     { "mcp-ns-a-id" → "customer-1",
                    └──────┬───────┘       "mcp-ns-b-id" → "customer-2" }
                           │ source_cluster="customer-1"
                           │ destination_cluster="customer-2"
                           ▼
                    ┌──────────────┐
                    │  Graph Node  │
                    │  cluster:    │
                    │  "customer-1"│ ── click → correct API call
                    └──────────────┘
```

## Backward Compatibility

All changes are **fully backward compatible**. Every new config defaults to nil/empty, preserving existing behavior:

| Change | Default (no config) | Effect on existing deployments |
|--------|--------------------|---------------------------------|
| `control_plane_namespace_selector` | nil → scan all namespaces | No change |
| `GetClustersForMesh()` | Returns all clusters for single mesh | No change |
| `IstioConfigCriteria.Cluster` | Empty → query all clusters | No change |
| `GetClusterNameMapping()` | nil → no translation | No change |
| `HandleClusters` mapping param | nil → pass-through | No change |

Standard deployments (single-cluster, primary-remote, multi-primary) where `CLUSTER_ID == cluster.Name` produce a nil mapping automatically. No configuration needed.

## Commits

| Commit | Description |
|--------|-------------|
| `1d763ee15` | Gap 1: `control_plane_namespace_selector` to scope istiod discovery |
| `351d20e62` | Fix: Handle `AccessibleNamespaceError` in multi-cluster loops |
| `db3527194` | Gap 3: Mesh-aware cluster scoping in business layer |
| `098dcd529` | Gap 5+6: Auto-discover metric cluster ID → name mapping |
