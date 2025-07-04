# Filter Services by Virtual Service Feature

## Overview

This feature adds a new behavior to Istio service discovery where sidecar proxies only discover services that are referenced as destinations in VirtualServices. When enabled, if no VirtualService is defined, no services will be exposed to the sidecar.

## Feature Flag

The feature is controlled by the `PILOT_FILTER_SERVICES_BY_VIRTUAL_SERVICE` environment variable:

```bash
export PILOT_FILTER_SERVICES_BY_VIRTUAL_SERVICE=true
```

## Default Behavior

By default, this feature is **disabled** (`false`). When disabled, Istio maintains its existing behavior:
- Sidecars discover and expose all Kubernetes destinations in the mesh
- All services exported to the namespace are visible to sidecars

## New Behavior (When Enabled)

When `PILOT_FILTER_SERVICES_BY_VIRTUAL_SERVICE=true`:
- Sidecars only discover services that are referenced as destinations in VirtualServices
- If no VirtualService exists, no services are exposed
- Services are filtered based on the destinations defined in HTTP, TCP, and TLS routes

## Use Cases

This feature is useful for:
1. **Large Meshes**: Reducing the number of services exposed to sidecars in large environments
2. **Security**: Only exposing services that are explicitly routed to via VirtualServices
3. **Performance**: Reducing the configuration size sent to proxies
4. **Compliance**: Ensuring only explicitly defined service routes are accessible

## Example

### Without Feature Flag (Default)
```yaml
# All services in the mesh are discovered by sidecars
# regardless of VirtualService definitions
```

### With Feature Flag Enabled
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews  # Only the 'reviews' service will be discovered
        subset: v1
    - destination:
        host: ratings  # Only the 'ratings' service will be discovered
```

In this example, when the feature flag is enabled, sidecars will only see the `reviews` and `ratings` services, not any other services in the mesh.

## Implementation Details

- The filtering is applied in the `DefaultSidecarScopeForNamespace` function
- Virtual service destinations are extracted using the existing `virtualServiceDestinations` function
- The feature maintains backward compatibility by being disabled by default
- Custom Sidecar resources are not affected by this feature flag

## Testing

Run the unit tests to verify the feature:

```bash
go test ./pilot/pkg/model -run TestFilterServicesByVirtualService -v
```