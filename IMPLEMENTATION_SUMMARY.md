# FilterServicesByVirtualService Feature Implementation Summary

## Overview

This PR implements a new experimental feature that changes Istio's default service discovery behavior. When enabled via the `PILOT_FILTER_SERVICES_BY_VIRTUAL_SERVICE` feature flag, sidecar proxies will only discover services that are explicitly referenced as destinations in VirtualServices.

## Key Changes

### 1. Feature Flag Addition
**File**: `pilot/pkg/features/pilot.go`

Added a new feature flag:
```go
FilterServicesByVirtualService = env.Register("PILOT_FILTER_SERVICES_BY_VIRTUAL_SERVICE", false,
    "If enabled, sidecar proxies will only discover services that are referenced as destinations in VirtualServices. "+
        "If no VirtualService is defined, no services will be exposed. This is an experimental feature that changes the default discovery behavior.").Get()
```

**Default**: `false` (maintains backward compatibility)

### 2. Core Logic Implementation
**File**: `pilot/pkg/model/sidecar.go`

Modified `DefaultSidecarScopeForNamespace` to conditionally filter services based on the feature flag:

```go
out.initFunc = sync.OnceFunc(func() {
    var services []*Service
    if features.FilterServicesByVirtualService {
        // When feature flag is enabled, only include services referenced in virtual services
        services = ps.getServicesReferencedInVirtualServices(configNamespace)
    } else {
        // Default behavior: include all services exported to this namespace
        services = ps.servicesExportedToNamespace(configNamespace)
    }
    // ... rest of initialization logic
})
```

**Integration with Lazy Initialization**: The feature properly integrates with Istio's new lazy sidecar scope initialization pattern introduced in the latest master.

### 3. Service Discovery Logic
**File**: `pilot/pkg/model/push_context.go`

Added `getServicesReferencedInVirtualServices` method that:

1. Gets all VirtualServices visible to the namespace
2. Extracts destination hosts from HTTP routes, TCP routes, and TLS routes
3. Matches referenced hosts against available services
4. Returns only services that are explicitly referenced

```go
func (ps *PushContext) getServicesReferencedInVirtualServices(configNamespace string) []*Service {
    // Get all virtual services visible to this namespace
    virtualServices := ps.VirtualServicesForGateway(configNamespace, constants.IstioMeshGateway)
    
    // If no virtual services, return empty list (core behavior change when flag is enabled)
    if len(virtualServices) == 0 {
        return []*Service{}
    }
    
    // Extract and match destination hosts
    // ... implementation details
}
```

### 4. Comprehensive Testing
**File**: `pilot/pkg/model/sidecar_test.go`

Added comprehensive tests covering all scenarios:

- ✅ **Feature disabled + VirtualServices present**: Should include all services (default behavior)
- ✅ **Feature enabled + VirtualServices present**: Should only include referenced services
- ✅ **Feature enabled + No VirtualServices**: Should include no services
- ✅ **Feature disabled + No VirtualServices**: Should include all services

## Behavior Analysis

### Default Behavior (Feature Flag = false)
- **Current Istio behavior is preserved**
- All services exported to the namespace are visible to sidecars
- No change in functionality

### New Behavior (Feature Flag = true)
- **Only services referenced in VirtualServices are exposed**
- If no VirtualServices exist, **no services are exposed**
- More restrictive and explicit service discovery

## Implementation Highlights

### ✅ Backward Compatibility
- Feature flag defaults to `false`
- Existing behavior is preserved when disabled
- No breaking changes for existing users

### ✅ Integration with Latest Master
- Successfully resolved merge conflicts with latest Istio master
- Adapted to new lazy sidecar scope initialization pattern
- Works with updated codebase architecture

### ✅ Comprehensive Testing
- All test cases pass
- Covers both enabled/disabled scenarios
- Tests edge cases (no VirtualServices)

### ✅ Performance Considerations
- Leverages existing VirtualService indexing
- Uses efficient host matching algorithms
- Integrates with lazy initialization for optimal performance

## Usage Example

### Enable the Feature
```bash
export PILOT_FILTER_SERVICES_BY_VIRTUAL_SERVICE=true
```

### Create VirtualService to Expose Services
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: expose-services
  namespace: default
spec:
  hosts:
  - details.default.svc.cluster.local
  http:
  - route:
    - destination:
        host: details.default.svc.cluster.local
      - destination:
          host: reviews.default.svc.cluster.local
```

**Result**: Only `details` and `reviews` services will be visible to sidecars in the `default` namespace.

## Testing Verification

All tests pass successfully:
```bash
$ go test ./pilot/pkg/model -run TestFilterServicesByVirtualService -v
=== RUN   TestFilterServicesByVirtualService
=== RUN   TestFilterServicesByVirtualService/Feature_disabled_-_should_include_all_services
=== RUN   TestFilterServicesByVirtualService/Feature_enabled_-_should_only_include_services_referenced_in_VS
=== RUN   TestFilterServicesByVirtualService/Feature_enabled,_no_VirtualServices_-_should_include_no_services
=== RUN   TestFilterServicesByVirtualService/Feature_disabled,_no_VirtualServices_-_should_include_all_services
--- PASS: TestFilterServicesByVirtualService (0.00s)
PASS
```

## Next Steps for Production

1. **Testing**: Extensive testing in development environments
2. **Documentation**: Update Istio documentation with feature details
3. **Migration Guide**: Provide guidance for users wanting to adopt this feature
4. **Monitoring**: Add metrics to monitor service discovery behavior
5. **Graduation**: Consider promoting from experimental to stable feature

## Files Modified

- `pilot/pkg/features/pilot.go` - Feature flag definition
- `pilot/pkg/model/sidecar.go` - Core logic implementation
- `pilot/pkg/model/push_context.go` - Service filtering logic
- `pilot/pkg/model/sidecar_test.go` - Comprehensive tests

## Risk Assessment

**Low Risk**: 
- Feature is disabled by default
- No impact on existing users
- Comprehensive test coverage
- Preserves all existing functionality when disabled