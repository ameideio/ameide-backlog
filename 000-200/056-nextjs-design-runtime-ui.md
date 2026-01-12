# 056: Next.js Design-Runtime UI Enhancement

## Overview

Enhance the existing Next.js model editor (`services/model-editor-react`) to support both design-time modeling and runtime monitoring. Replace the custom API client with the enhanced Ameide SDK and create a dual-purpose UI for the complete workflows lifecycle.

## Goals

- Replace custom API client with Ameide SDK
- Add runtime monitoring capabilities
- Create tabbed interface for design/runtime views
- Support real-time updates via WebSocket
- Integrate all design services (BPMN, ArchiMate, WfSpec)
- Enable seamless design-to-runtime workflows

## Current State

**Existing React app has:**
- ✅ BPMN editor integration (bpmn-js)
- ✅ Document list and diagram viewing
- ✅ SDK integration with design.bpmn namespace
- ✅ React Query for data fetching
- ✅ Zustand for state management
- ✅ Bootstrap UI components
- ✅ TypeScript types from proto definitions

**Missing:**
- ❌ No runtime monitoring features
- ❌ No ArchiMate support
- ❌ No real-time updates (WebSocket ready in gateway)
- ❌ Limited to BPMN editing only

**Available now:**
- ✅ BPMN service with full REST/gRPC API
- ✅ WebSocket support in gateway service
- ✅ Authentication via JWT tokens

## Proposed Enhancement

### 1. SDK Integration

```typescript
// src/lib/sdk-client.ts
import { AmeideClient } from '@ameide/sdk';
import { QueryClient } from '@tanstack/react-query';

export const ameideClient = new AmeideClient({
  baseUrl: process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000',
  auth: {
    type: 'bearer',  // JWT auth implemented in gateway
    token: getAuthToken(), // Get from auth context/localStorage
  },
});

// React Query integration
export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      queryFn: async ({ queryKey }) => {
        const [service, method, ...params] = queryKey;
        return ameideClient[service][method](...params);
      },
    },
  },
});
```

### 2. UI Structure

```
src/
├── app/
│   ├── design/                 # Design-time views
│   │   ├── bpmn/              # BPMN editor ✅ Can be implemented now
│   │   └── archimate/         # ArchiMate viewer (pending service)
│   ├── runtime/               # Runtime monitoring
│   │   ├── workflows/         # Workflow executions
│   │   ├── agents/            # Agent monitoring
│   │   └── ipa/               # IPA dashboard
│   └── layout.tsx             # Main layout with tabs
├── components/
│   ├── design/
│   │   ├── BPMNEditor/        # Existing, updated
│   │   ├── ArchiMateViewer/   # New
│   │   └── WfSpecEditor/      # New
│   ├── runtime/
│   │   ├── WorkflowMonitor/   # New
│   │   ├── AgentDashboard/    # New
│   │   └── IPAExecution/      # New
│   └── shared/
│       ├── ModelSelector/     # Cross-cutting
│       └── RealtimeStatus/    # WebSocket status
```

### 3. Design Tab Features

```typescript
// components/design/ModelWorkspace.tsx
export function ModelWorkspace() {
  const [activeModel, setActiveModel] = useState(null);
  const [modelType, setModelType] = useState<'bpmn' | 'archimate' | 'wfspec'>('bpmn');
  
  // Query models
  const { data: models } = useQuery({
    queryKey: ['design', modelType, 'list'],
    queryFn: () => ameideClient.design[modelType].list(),
  });
  
  // Deploy to runtime
  const deployMutation = useMutation({
    mutationFn: async (modelId: string) => {
      return ameideClient.design.transformations.convertToWorkflow({
        modelId,
        modelType,
      });
    },
    onSuccess: (result) => {
      toast.success(`Deployed as workflows: ${result.workflowsId}`);
      router.push(`/runtime/workflows/${result.workflowsId}`);
    },
  });
  
  return (
    <div className="model-workspace">
      <ModelTypeSelector value={modelType} onChange={setModelType} />
      <ModelList models={models} onSelect={setActiveModel} />
      <ModelEditor model={activeModel} type={modelType} />
      <DeployButton onClick={() => deployMutation.mutate(activeModel.id)} />
    </div>
  );
}
```

### 4. Runtime Tab Features

```typescript
// components/runtime/ExecutionMonitor.tsx
export function ExecutionMonitor() {
  const [executions, setExecutions] = useState([]);
  
  // Real-time updates
  useEffect(() => {
    const subscription = ameideClient.runtime.workflows
      .subscribeToExecutions()
      .subscribe((event) => {
        setExecutions(prev => updateExecution(prev, event));
      });
      
    return () => subscription.unsubscribe();
  }, []);
  
  return (
    <div className="execution-monitor">
      <ExecutionList executions={executions} />
      <ExecutionDetails selected={selectedExecution} />
      <LogStream executionId={selectedExecution?.id} />
    </div>
  );
}
```

### 5. WebSocket Integration

```typescript
// hooks/useRealtimeUpdates.ts
export function useRealtimeUpdates(resourceType: string, resourceId: string) {
  const [events, setEvents] = useState([]);
  const [status, setStatus] = useState<'connected' | 'disconnected'>('disconnected');
  
  useEffect(() => {
    // Update (648): do not rely on NEXT_PUBLIC_* runtime URL knobs in cluster deployments.
    // Use same-origin websocket endpoints derived at runtime.
    const wsOrigin = window.location.origin.replace(/^http/, 'ws');
    const ws = new WebSocket(`${wsOrigin}/ws`);
    
    ws.onopen = () => {
      setStatus('connected');
      ws.send(JSON.stringify({
        action: 'subscribe',
        resourceType,
        resourceId,
      }));
    };
    
    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      setEvents(prev => [...prev, data]);
    };
    
    return () => ws.close();
  }, [resourceType, resourceId]);
  
  return { events, status };
}
```

### 6. Navigation Structure

```typescript
// app/layout.tsx
export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <nav className="navbar">
          <NavTabs>
            <NavTab href="/design" icon="pencil">Design</NavTab>
            <NavTab href="/runtime" icon="play">Runtime</NavTab>
            <NavTab href="/analytics" icon="chart">Analytics</NavTab>
          </NavTabs>
        </nav>
        <main>{children}</main>
      </body>
    </html>
  );
}
```

## Success Criteria

- [x] SDK fully integrated, custom client removed
- [x] Design tab supports BPMN (ArchiMate when ready)
- [ ] Runtime tab shows live executions
- [ ] WebSocket updates working
- [ ] Seamless design-to-runtime deployment
- [x] Existing BPMN editor functionality preserved
- [ ] Performance: <100ms tab switches

## Current Progress

- ✅ BPMN API available via REST in gateway
- ✅ WebSocket infrastructure ready in gateway
- ✅ JWT authentication implemented
- ✅ SDK updated with design services (BPMN namespace added)
- ✅ UI migrated to use Ameide SDK (custom API client removed)
- ✅ Model editor renamed to www-ameide-portal-canvas
- ✅ TypeScript types aligned with proto definitions
- ✅ React Query hooks integrated with SDK

## Dependencies

- #055 Enhanced SDK with design services ✅ DONE
- #054 Gateway with WebSocket support ✅ DONE
- BPMN proto service ✅ DONE
- Other design services ❌ NOT STARTED
- Runtime services ⚠️ PARTIALLY IMPLEMENTED

## Estimated Effort

- SDK integration: ✅ COMPLETED
- Runtime components: 3 days
- WebSocket setup: 2 days (infrastructure ready)
- UI reorganization: 2 days
- Testing: 2 days
- Documentation: 1 day
- **Total: 10 days remaining**

## Next Steps

1. ~~Wait for SDK enhancement (#055) to complete~~ ✅ DONE
2. ~~Start with BPMN design tab using SDK~~ ✅ DONE
3. Implement WebSocket connection to gateway
4. Add runtime monitoring for implemented services
5. Implement ArchiMate and WfSpec support when services are ready
6. Add real-time updates for collaborative editing

## Risks

- SDK migration complexity
- WebSocket connection stability
- State management complexity
- Performance with many models
- Browser compatibility
