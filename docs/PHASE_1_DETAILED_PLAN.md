# Phase 1: Core Architecture - Detailed Implementation Plan

## Executive Summary

Phase 1 establishes the foundational architecture of n9n, focusing on the core systems that enable reactive workflow automation. This phase delivers the essential building blocks: Entity Component System (ECS) foundation, reactive stream system, basic execution engine, and TypeScript workflow DSL. Together, these components form a unique architecture that combines high-performance ECS patterns with reactive programming principles.

**Duration**: 12-16 weeks  
**Team Size**: 4-6 engineers  
**Key Deliverable**: Functional reactive workflow execution engine with TypeScript DSL

---

## 1. ECS Foundation Implementation

### 1.1 Core ECS Architecture

#### 1.1.1 Entity System Design

```typescript
// Core entity management system
interface Entity {
  id: string;
  components: Map<ComponentType, Component>;
  archetype: ArchetypeId;
  generation: number;
}

interface World {
  entities: SparseSet<Entity>;
  archetypes: Map<ArchetypeId, Archetype>;
  systems: System[];
  resources: Map<ResourceType, Resource>;
}

class ECSWorld implements World {
  private entityIdGenerator: IdGenerator;
  private componentStorage: ComponentStorage;
  private queryCache: QueryCache;
  
  // High-performance entity creation with archetype optimization
  createEntity(components: Component[]): EntityId;
  
  // Bulk component operations for performance
  addComponents(entityId: EntityId, components: Component[]): void;
  removeComponents(entityId: EntityId, componentTypes: ComponentType[]): void;
  
  // Optimized querying with archetype filtering
  query<T extends Component[]>(...componentTypes: ComponentType[]): Query<T>;
}
```

#### 1.1.2 Component Definitions for Workflow Nodes

```typescript
// Core workflow components
interface NodeInfoComponent extends Component {
  type: 'NodeInfo';
  uuid: string;
  displayName: string;
  description?: string;
  version: string;
  category: NodeCategory;
}

interface PositionComponent extends Component {
  type: 'Position';
  x: number;
  y: number;
  z?: number; // For 3D layouts in future
}

interface SlotMetadataComponent extends Component {
  type: 'SlotMetadata';
  inputs: Map<string, SlotDefinition>;
  outputs: Map<string, SlotDefinition>;
}

interface ReactiveSlotComponent extends Component {
  type: 'ReactiveSlot';
  slotId: string;
  nodeId: string;
  slotType: 'input' | 'output';
  dataType: 'stream' | 'object' | 'primitive';
  reactive: boolean;
  
  // Stream-specific properties
  emissionRate?: number;
  subscriberCount?: number;
  bufferSize?: number;
  backpressureStrategy?: BackpressureStrategy;
}

interface ExecutionStateComponent extends Component {
  type: 'ExecutionState';
  status: 'idle' | 'running' | 'completed' | 'error' | 'paused';
  startedAt?: number;
  completedAt?: number;
  error?: Error;
  metrics: {
    executionCount: number;
    averageExecutionTime: number;
    totalExecutionTime: number;
  };
}

interface ConnectionComponent extends Component {
  type: 'Connection';
  sourceEntity: EntityId;
  sourceSlot: string;
  targetEntity: EntityId;
  targetSlot: string;
  
  // Connection configuration
  config: {
    transform?: TransformFunction;
    filter?: FilterFunction;
    backpressure?: BackpressureConfig;
    errorHandling?: ErrorHandlingConfig;
    rateLimit?: RateLimitConfig;
  };
  
  // Runtime state
  state: {
    dataFlowRate: number;
    errorCount: number;
    totalTransferred: number;
    lastActivity: number;
  };
}
```

#### 1.1.3 System Architecture

```typescript
// Core ECS systems for workflow execution
abstract class System {
  abstract readonly priority: number;
  abstract readonly requiredComponents: ComponentType[];
  
  abstract execute(world: World, deltaTime: number): void;
  
  // Lifecycle hooks
  onSystemAdded?(world: World): void;
  onSystemRemoved?(world: World): void;
}

class ReactiveSlotSystem extends System {
  readonly priority = 100;
  readonly requiredComponents = [ReactiveSlotComponent, SlotMetadataComponent];
  
  execute(world: World, deltaTime: number): void {
    // Update reactive slot states
    // Handle stream emissions
    // Manage backpressure
    // Update metrics
  }
}

class ConnectionSystem extends System {
  readonly priority = 90;
  readonly requiredComponents = [ConnectionComponent];
  
  execute(world: World, deltaTime: number): void {
    // Process data flow through connections
    // Apply transforms and filters
    // Handle errors and retries
    // Update connection metrics
  }
}

class ExecutionSystem extends System {
  readonly priority = 80;
  readonly requiredComponents = [ExecutionStateComponent, NodeInfoComponent];
  
  execute(world: World, deltaTime: number): void {
    // Execute node logic
    // Update execution states
    // Handle node lifecycle
    // Collect execution metrics
  }
}
```

**Milestone 1.1**: ECS Foundation (Week 2)
- ✅ Core ECS world implementation
- ✅ Component type system
- ✅ Basic system execution loop
- ✅ Entity archetype optimization
- ✅ Performance benchmarks (>100k entities/frame)

### 1.2 Advanced ECS Features

#### 1.2.1 Archetype-Based Storage

```typescript
// Optimized storage for workflow performance
interface Archetype {
  id: ArchetypeId;
  componentTypes: Set<ComponentType>;
  entities: EntityId[];
  componentArrays: Map<ComponentType, ComponentArray>;
}

class ArchetypeManager {
  private archetypes: Map<ArchetypeId, Archetype>;
  private entityToArchetype: Map<EntityId, ArchetypeId>;
  
  // Fast archetype transitions when components change
  moveEntity(entityId: EntityId, newComponentTypes: Set<ComponentType>): void;
  
  // Optimized queries using archetype matching
  findMatchingArchetypes(queryComponents: ComponentType[]): Archetype[];
}
```

#### 1.2.2 Query Optimization

```typescript
class QueryCache {
  private cache: Map<QuerySignature, CachedQuery>;
  
  // Cached queries for repeated operations
  getCachedQuery<T extends Component[]>(
    ...componentTypes: ComponentType[]
  ): Query<T> {
    const signature = this.createSignature(componentTypes);
    
    if (!this.cache.has(signature)) {
      this.cache.set(signature, this.buildQuery(componentTypes));
    }
    
    return this.cache.get(signature) as Query<T>;
  }
  
  // Incremental query updates when archetypes change
  invalidateQueriesForArchetype(archetypeId: ArchetypeId): void;
}
```

**Milestone 1.2**: Advanced ECS Features (Week 4)
- ✅ Archetype-based component storage
- ✅ Query optimization and caching
- ✅ Component serialization system
- ✅ ECS performance profiling tools

---

## 2. Reactive Stream System (Built on RxJS)

### 2.1 RxJS Foundation with Workflow Extensions

#### 2.1.1 RxJS-Based Stream Implementation

```typescript
import { Observable, Subject, Subscription, Scheduler, Observer } from 'rxjs';
import { 
  map, filter, buffer, debounceTime, retry, catchError, 
  share, shareReplay, takeUntil, mergeMap, switchMap
} from 'rxjs/operators';

// Extend RxJS with workflow-specific features
interface WorkflowObservable<T> extends Observable<T> {
  // Workflow-specific backpressure handling
  withBackpressure(strategy: BackpressureStrategy): WorkflowObservable<T>;
  
  // ECS-aware error handling
  catchWorkflowError(handler: WorkflowErrorHandler<T>): WorkflowObservable<T>;
  
  // Workflow-specific retry with exponential backoff
  retryWithBackoff(config: RetryConfig): WorkflowObservable<T>;
  
  // Connection health monitoring
  withHealthMonitoring(config: HealthConfig): WorkflowObservable<T>;
  
  // Workflow metrics collection
  collectMetrics(metricsCollector: MetricsCollector): WorkflowObservable<T>;
}

// Workflow-enhanced stream implementation
class WorkflowObservableImpl<T> extends Observable<T> implements WorkflowObservable<T> {
  constructor(
    subscribe: (observer: Observer<T>) => Subscription,
    private workflowContext?: WorkflowContext
  ) {
    super(subscribe);
  }
  
  withBackpressure(strategy: BackpressureStrategy): WorkflowObservable<T> {
    return new WorkflowObservableImpl<T>(observer => {
      const backpressureBuffer = new BackpressureBuffer<T>(strategy);
      
      return this.subscribe({
        next: (value) => {
          if (backpressureBuffer.shouldBuffer(value)) {
            backpressureBuffer.add(value);
          } else {
            observer.next(value);
          }
        },
        error: (err) => observer.error(err),
        complete: () => {
          // Flush remaining buffer
          backpressureBuffer.flush().forEach(v => observer.next(v));
          observer.complete();
        }
      });
    }, this.workflowContext);
  }
  
  catchWorkflowError(handler: WorkflowErrorHandler<T>): WorkflowObservable<T> {
    return new WorkflowObservableImpl<T>(observer => {
      return this.pipe(
        catchError((error: any, caught: Observable<T>) => {
          const recovery = handler.handleError(error, this.workflowContext);
          
          switch (recovery.action) {
            case 'retry':
              return caught.pipe(retry(recovery.retries || 3));
            case 'fallback':
              return recovery.fallbackValue ? 
                Observable.of(recovery.fallbackValue) : 
                Observable.empty();
            case 'rethrow':
            default:
              throw error;
          }
        })
      ).subscribe(observer);
    }, this.workflowContext);
  }
  
  retryWithBackoff(config: RetryConfig): WorkflowObservable<T> {
    return new WorkflowObservableImpl<T>(observer => {
      return this.pipe(
        retry({
          count: config.maxRetries,
          delay: (error, retryCount) => {
            const delay = Math.min(
              config.baseDelay * Math.pow(2, retryCount - 1),
              config.maxDelay || 30000
            );
            return timer(delay);
          }
        })
      ).subscribe(observer);
    }, this.workflowContext);
  }
}
```

#### 2.1.2 Workflow-Specific RxJS Operators

```typescript
import { Observable, OperatorFunction, timer, EMPTY, of } from 'rxjs';
import { 
  map, filter, buffer, debounceTime, retry, catchError, 
  mergeMap, concatMap, share, shareReplay, tap, 
  windowTime, windowCount, throttleTime
} from 'rxjs/operators';

// Workflow-specific operators built on RxJS
namespace WorkflowOperators {
  
  // Enhanced rate limiting with burst handling
  export function workflowRateLimit<T>(config: RateLimitConfig): OperatorFunction<T, T> {
    return (source: Observable<T>) => {
      let tokens = config.burstSize || config.requestsPerWindow;
      let lastRefill = Date.now();
      
      return source.pipe(
        mergeMap(value => {
          const now = Date.now();
          const timePassed = now - lastRefill;
          const tokensToAdd = Math.floor(timePassed / config.windowMs * config.requestsPerWindow);
          
          tokens = Math.min(tokens + tokensToAdd, config.burstSize || config.requestsPerWindow);
          lastRefill = now;
          
          if (tokens > 0) {
            tokens--;
            return of(value);
          } else {
            // Apply backpressure strategy
            switch (config.backpressureStrategy) {
              case 'drop':
                return EMPTY;
              case 'delay':
                const delayTime = config.windowMs / config.requestsPerWindow;
                return timer(delayTime).pipe(map(() => value));
              default:
                return of(value);
            }
          }
        })
      );
    };
  }
  
  // Workflow-aware circuit breaker
  export function circuitBreaker<T>(config: CircuitBreakerConfig): OperatorFunction<T, T> {
    return (source: Observable<T>) => {
      let state: 'closed' | 'open' | 'half-open' = 'closed';
      let errorCount = 0;
      let lastFailureTime = 0;
      
      return source.pipe(
        mergeMap(value => {
          if (state === 'open') {
            if (Date.now() - lastFailureTime > config.resetTimeoutMs) {
              state = 'half-open';
            } else {
              throw new Error('Circuit breaker is open');
            }
          }
          
          return of(value).pipe(
            tap({
              next: () => {
                if (state === 'half-open') {
                  state = 'closed';
                  errorCount = 0;
                }
              },
              error: (error) => {
                errorCount++;
                lastFailureTime = Date.now();
                
                if (errorCount >= config.errorThreshold) {
                  state = 'open';
                }
                
                throw error;
              }
            })
          );
        })
      );
    };
  }
  
  // Advanced windowing for session-based aggregation
  export function sessionWindow<T>(
    sessionTimeoutMs: number,
    keySelector: (value: T) => string
  ): OperatorFunction<T, T[]> {
    return (source: Observable<T>) => {
      const sessions = new Map<string, { 
        items: T[], 
        lastActivity: number,
        subject: Subject<T[]>
      }>();
      
      return new Observable(observer => {
        const cleanup = () => {
          sessions.forEach(session => session.subject.complete());
          sessions.clear();
        };
        
        const subscription = source.subscribe({
          next: (value) => {
            const key = keySelector(value);
            const now = Date.now();
            
            // Clean up expired sessions
            for (const [sessionKey, session] of sessions.entries()) {
              if (now - session.lastActivity > sessionTimeoutMs) {
                session.subject.next([...session.items]);
                session.subject.complete();
                sessions.delete(sessionKey);
              }
            }
            
            // Add to existing session or create new one
            let session = sessions.get(key);
            if (!session) {
              session = {
                items: [],
                lastActivity: now,
                subject: new Subject<T[]>()
              };
              sessions.set(key, session);
              
              // Subscribe to session completion
              session.subject.subscribe({
                next: (items) => observer.next(items),
                complete: () => sessions.delete(key)
              });
            }
            
            session.items.push(value);
            session.lastActivity = now;
          },
          error: (err) => {
            cleanup();
            observer.error(err);
          },
          complete: () => {
            // Emit all remaining sessions
            sessions.forEach(session => {
              if (session.items.length > 0) {
                session.subject.next([...session.items]);
              }
              session.subject.complete();
            });
            cleanup();
            observer.complete();
          }
        });
        
        return () => {
          subscription.unsubscribe();
          cleanup();
        };
      });
    };
  }
  
  // Fan-out with conditional routing
  export function fanOut<T>(
    routes: Array<{
      condition: (value: T) => boolean;
      transform?: (value: T) => any;
      target: Subject<any>;
    }>
  ): OperatorFunction<T, T> {
    return (source: Observable<T>) => {
      return source.pipe(
        tap(value => {
          routes.forEach(route => {
            if (route.condition(value)) {
              const outputValue = route.transform ? route.transform(value) : value;
              route.target.next(outputValue);
            }
          });
        })
      );
    };
  }
  
  // Metrics collection operator
  export function collectMetrics<T>(
    metricsCollector: MetricsCollector,
    metricName: string
  ): OperatorFunction<T, T> {
    return (source: Observable<T>) => {
      let count = 0;
      let startTime = Date.now();
      
      return source.pipe(
        tap({
          next: (value) => {
            count++;
            metricsCollector.increment(`${metricName}.count`);
            metricsCollector.gauge(`${metricName}.rate`, count / ((Date.now() - startTime) / 1000));
          },
          error: (error) => {
            metricsCollector.increment(`${metricName}.errors`);
          }
        })
      );
    };
  }
}
```

#### 2.1.3 Backpressure Management

```typescript
// Sophisticated backpressure handling for workflow data flows
interface BackpressureStrategy {
  strategy: 'buffer' | 'drop' | 'latest' | 'block';
  bufferSize?: number;
  timeoutMs?: number;
  onOverflow?: (droppedValue: any) => void;
}

class BackpressureManager {
  private strategies: Map<string, BackpressureStrategy> = new Map();
  
  registerStrategy(name: string, strategy: BackpressureStrategy): void {
    this.strategies.set(name, strategy);
  }
  
  applyBackpressure<T>(
    stream: ReactiveStream<T>,
    strategyName: string
  ): ReactiveStream<T> {
    const strategy = this.strategies.get(strategyName);
    if (!strategy) {
      throw new Error(`Unknown backpressure strategy: ${strategyName}`);
    }
    
    return stream.withBackpressure(strategy);
  }
}

// Circuit breaker pattern for stream resilience
class StreamCircuitBreaker<T> {
  private state: 'closed' | 'open' | 'half-open' = 'closed';
  private errorCount = 0;
  private lastFailureTime = 0;
  
  constructor(
    private config: {
      errorThreshold: number;
      timeoutMs: number;
      resetTimeoutMs: number;
    }
  ) {}
  
  execute(operation: () => T): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailureTime > this.config.resetTimeoutMs) {
        this.state = 'half-open';
      } else {
        throw new Error('Circuit breaker is open');
      }
    }
    
    return this.performOperation(operation);
  }
}
```

**Milestone 2.1**: Core Stream Infrastructure (Week 6)
- ✅ Reactive stream implementation
- ✅ Core stream operators library
- ✅ Backpressure management system
- ✅ Circuit breaker implementation
- ✅ Stream performance benchmarks

### 2.2 Workflow-Specific Stream Features

#### 2.2.1 Slot-to-Slot Stream Connections

```typescript
// Integration between ECS and reactive streams
class SlotStreamManager {
  private streamConnections: Map<ConnectionId, StreamConnection> = new Map();
  private slotStreams: Map<SlotId, ReactiveStream<any>> = new Map();
  
  createSlotStream<T>(slotId: SlotId, config: SlotStreamConfig): ReactiveStream<T> {
    const stream = new ReactiveStreamImpl<T>(
      new SlotStreamSource(slotId),
      config.scheduler
    );
    
    if (config.backpressure) {
      stream.withBackpressure(config.backpressure);
    }
    
    this.slotStreams.set(slotId, stream);
    return stream;
  }
  
  connectSlots(
    sourceSlot: SlotId,
    targetSlot: SlotId,
    config: ConnectionConfig
  ): ConnectionId {
    const sourceStream = this.slotStreams.get(sourceSlot);
    const targetStream = this.slotStreams.get(targetSlot);
    
    if (!sourceStream || !targetStream) {
      throw new Error('Source or target slot stream not found');
    }
    
    let pipeline = sourceStream;
    
    // Apply transformations
    if (config.transform) {
      pipeline = pipeline.pipe(StreamOperators.map(config.transform));
    }
    
    // Apply filters
    if (config.filter) {
      pipeline = pipeline.pipe(StreamOperators.filter(config.filter));
    }
    
    // Apply rate limiting
    if (config.rateLimit) {
      pipeline = pipeline.pipe(StreamOperators.rateLimit(config.rateLimit));
    }
    
    // Connect to target
    const subscription = pipeline.subscribe({
      next: (value) => targetStream.emit(value),
      error: (error) => this.handleConnectionError(error, config),
      complete: () => {}
    });
    
    const connectionId = generateConnectionId();
    this.streamConnections.set(connectionId, {
      id: connectionId,
      sourceSlot,
      targetSlot,
      subscription,
      config
    });
    
    return connectionId;
  }
}
```

#### 2.2.2 Advanced Stream Patterns

```typescript
// Workflow-specific stream patterns
class WorkflowStreamPatterns {
  
  // Fan-out pattern: One source to multiple targets
  static createFanOut<T>(
    source: ReactiveStream<T>,
    targets: Array<{
      stream: ReactiveStream<T>;
      filter?: (value: T) => boolean;
      transform?: (value: T) => any;
    }>
  ): Subscription[] {
    return targets.map(target => {
      let pipeline = source;
      
      if (target.filter) {
        pipeline = pipeline.pipe(StreamOperators.filter(target.filter));
      }
      
      if (target.transform) {
        pipeline = pipeline.pipe(StreamOperators.map(target.transform));
      }
      
      return pipeline.subscribe({
        next: (value) => target.stream.emit(value),
        error: (error) => target.stream.error(error),
        complete: () => target.stream.complete()
      });
    });
  }
  
  // Fan-in pattern: Multiple sources to one target
  static createFanIn<T>(
    sources: Array<{
      stream: ReactiveStream<T>;
      priority?: number;
      tag?: string;
    }>,
    target: ReactiveStream<T>,
    config: {
      mergeStrategy: 'priority' | 'round-robin' | 'timestamp';
      bufferSize?: number;
    }
  ): Subscription[] {
    const merger = new StreamMerger(sources, target, config);
    return merger.connect();
  }
  
  // Pipeline pattern: Chained transformations
  static createPipeline<T>(
    source: ReactiveStream<T>,
    stages: Array<{
      transform?: (value: any) => any;
      filter?: (value: any) => boolean;
      validate?: (value: any) => boolean;
    }>
  ): ReactiveStream<any> {
    return stages.reduce((stream, stage) => {
      let result = stream;
      
      if (stage.filter) {
        result = result.pipe(StreamOperators.filter(stage.filter));
      }
      
      if (stage.transform) {
        result = result.pipe(StreamOperators.map(stage.transform));
      }
      
      if (stage.validate) {
        result = result.pipe(StreamOperators.filter(stage.validate));
      }
      
      return result;
    }, source);
  }
}
```

**Milestone 2.2**: Workflow Stream Features (Week 8)
- ✅ Slot-to-slot stream connections
- ✅ Advanced stream patterns (fan-out, fan-in, pipeline)
- ✅ Stream error handling and recovery
- ✅ Stream metrics and monitoring

---

## 3. Basic Execution Engine

### 3.1 Workflow Execution Core

#### 3.1.1 Execution Engine Architecture

```typescript
// Core workflow execution engine
interface WorkflowExecutor {
  executeWorkflow(workflow: WorkflowDefinition): Promise<ExecutionResult>;
  pauseExecution(workflowId: string): Promise<void>;
  resumeExecution(workflowId: string): Promise<void>;
  stopExecution(workflowId: string): Promise<void>;
  getExecutionStatus(workflowId: string): ExecutionStatus;
}

class ReactiveWorkflowExecutor implements WorkflowExecutor {
  private ecsWorld: ECSWorld;
  private streamManager: SlotStreamManager;
  private executionContexts: Map<string, ExecutionContext>;
  private scheduler: WorkflowScheduler;
  
  constructor() {
    this.ecsWorld = new ECSWorld();
    this.streamManager = new SlotStreamManager();
    this.scheduler = new WorkflowScheduler();
    this.initializeSystems();
  }
  
  async executeWorkflow(workflow: WorkflowDefinition): Promise<ExecutionResult> {
    const context = this.createExecutionContext(workflow);
    
    try {
      // 1. Build ECS entities from workflow definition
      const entities = await this.buildWorkflowEntities(workflow);
      
      // 2. Initialize reactive streams
      await this.initializeStreams(entities);
      
      // 3. Start execution
      const result = await this.startExecution(context);
      
      return result;
      
    } catch (error) {
      return this.handleExecutionError(context, error);
    }
  }
  
  private async buildWorkflowEntities(workflow: WorkflowDefinition): Promise<EntityId[]> {
    const entities: EntityId[] = [];
    
    // Create node entities
    for (const node of workflow.nodes) {
      const entity = this.ecsWorld.createEntity([
        new NodeInfoComponent({
          uuid: node.uuid,
          displayName: node.displayName,
          version: node.version
        }),
        new PositionComponent({
          x: node.position.x,
          y: node.position.y
        }),
        new SlotMetadataComponent({
          inputs: new Map(Object.entries(node.inputSlots || {})),
          outputs: new Map(Object.entries(node.outputSlots || {}))
        }),
        new ExecutionStateComponent({
          status: 'idle',
          metrics: {
            executionCount: 0,
            averageExecutionTime: 0,
            totalExecutionTime: 0
          }
        })
      ]);
      
      entities.push(entity);
    }
    
    // Create connection entities
    for (const connection of workflow.connections) {
      const entity = this.ecsWorld.createEntity([
        new ConnectionComponent({
          sourceEntity: this.findEntityByNodeId(connection.from.nodeId),
          sourceSlot: connection.from.slot,
          targetEntity: this.findEntityByNodeId(connection.to.nodeId),
          targetSlot: connection.to.slot,
          config: connection.config || {},
          state: {
            dataFlowRate: 0,
            errorCount: 0,
            totalTransferred: 0,
            lastActivity: Date.now()
          }
        })
      ]);
      
      entities.push(entity);
    }
    
    return entities;
  }
  
  private async initializeStreams(entities: EntityId[]): Promise<void> {
    // Create reactive streams for all slots
    const slotQuery = this.ecsWorld.query(SlotMetadataComponent);
    
    for (const [entity, slotMetadata] of slotQuery) {
      // Create input streams
      for (const [slotName, slotDef] of slotMetadata.inputs) {
        if (slotDef.reactive) {
          const slotId = `${entity}:${slotName}`;
          this.streamManager.createSlotStream(slotId, {
            backpressure: slotDef.backpressure,
            scheduler: this.scheduler.getSlotScheduler(slotId)
          });
        }
      }
      
      // Create output streams
      for (const [slotName, slotDef] of slotMetadata.outputs) {
        if (slotDef.reactive) {
          const slotId = `${entity}:${slotName}`;
          this.streamManager.createSlotStream(slotId, {
            backpressure: slotDef.backpressure,
            scheduler: this.scheduler.getSlotScheduler(slotId)
          });
        }
      }
    }
    
    // Connect streams based on connections
    const connectionQuery = this.ecsWorld.query(ConnectionComponent);
    
    for (const [entity, connection] of connectionQuery) {
      const sourceSlotId = `${connection.sourceEntity}:${connection.sourceSlot}`;
      const targetSlotId = `${connection.targetEntity}:${connection.targetSlot}`;
      
      this.streamManager.connectSlots(sourceSlotId, targetSlotId, connection.config);
    }
  }
}
```

#### 3.1.2 Dependency Resolution Engine

```typescript
// Sophisticated dependency resolution for reactive workflows
class DependencyResolver {
  
  resolveExecutionOrder(workflow: WorkflowDefinition): ExecutionPlan {
    const graph = this.buildDependencyGraph(workflow);
    const order = this.topologicalSort(graph);
    
    return {
      executionOrder: order,
      parallelGroups: this.identifyParallelGroups(order, graph),
      criticalPath: this.findCriticalPath(graph),
      cyclicDependencies: this.detectCycles(graph)
    };
  }
  
  private buildDependencyGraph(workflow: WorkflowDefinition): DependencyGraph {
    const graph = new Map<string, Set<string>>();
    
    // Initialize nodes
    for (const node of workflow.nodes) {
      graph.set(node.id, new Set());
    }
    
    // Add dependencies based on connections
    for (const connection of workflow.connections) {
      const source = connection.from.nodeId;
      const target = connection.to.nodeId;
      
      // Target depends on source
      graph.get(target)?.add(source);
    }
    
    return graph;
  }
  
  private topologicalSort(graph: DependencyGraph): string[] {
    const visited = new Set<string>();
    const visiting = new Set<string>();
    const result: string[] = [];
    
    const visit = (nodeId: string) => {
      if (visiting.has(nodeId)) {
        throw new Error(`Circular dependency detected involving node: ${nodeId}`);
      }
      
      if (visited.has(nodeId)) {
        return;
      }
      
      visiting.add(nodeId);
      
      const dependencies = graph.get(nodeId) || new Set();
      for (const dep of dependencies) {
        visit(dep);
      }
      
      visiting.delete(nodeId);
      visited.add(nodeId);
      result.push(nodeId);
    };
    
    for (const nodeId of graph.keys()) {
      if (!visited.has(nodeId)) {
        visit(nodeId);
      }
    }
    
    return result.reverse();
  }
  
  private identifyParallelGroups(
    executionOrder: string[],
    graph: DependencyGraph
  ): string[][] {
    const groups: string[][] = [];
    const processed = new Set<string>();
    
    for (const nodeId of executionOrder) {
      if (processed.has(nodeId)) continue;
      
      const parallelNodes = this.findParallelNodes(nodeId, graph, processed);
      groups.push(parallelNodes);
      
      for (const node of parallelNodes) {
        processed.add(node);
      }
    }
    
    return groups;
  }
}
```

#### 3.1.3 Execution Context Management

```typescript
// Rich execution context for workflow runs
interface ExecutionContext {
  workflowId: string;
  executionId: string;
  startTime: number;
  status: ExecutionStatus;
  
  // ECS world state
  ecsWorld: ECSWorld;
  
  // Stream management
  streamManager: SlotStreamManager;
  activeStreams: Map<string, ReactiveStream<any>>;
  
  // Error handling
  errors: ExecutionError[];
  errorHandlers: Map<string, ErrorHandler>;
  
  // Metrics and monitoring
  metrics: ExecutionMetrics;
  profiler: ExecutionProfiler;
  
  // Resource management
  resources: Map<string, any>;
  cleanup: (() => Promise<void>)[];
}

class ExecutionContextManager {
  private contexts: Map<string, ExecutionContext> = new Map();
  
  createContext(workflow: WorkflowDefinition): ExecutionContext {
    const context: ExecutionContext = {
      workflowId: workflow.id,
      executionId: generateExecutionId(),
      startTime: Date.now(),
      status: 'initializing',
      ecsWorld: new ECSWorld(),
      streamManager: new SlotStreamManager(),
      activeStreams: new Map(),
      errors: [],
      errorHandlers: new Map(),
      metrics: new ExecutionMetrics(),
      profiler: new ExecutionProfiler(),
      resources: new Map(),
      cleanup: []
    };
    
    this.contexts.set(context.executionId, context);
    return context;
  }
  
  async cleanupContext(executionId: string): Promise<void> {
    const context = this.contexts.get(executionId);
    if (!context) return;
    
    // Run cleanup tasks
    for (const cleanup of context.cleanup) {
      try {
        await cleanup();
      } catch (error) {
        console.error('Cleanup error:', error);
      }
    }
    
    // Dispose ECS world
    context.ecsWorld.dispose();
    
    // Close all streams
    for (const stream of context.activeStreams.values()) {
      stream.complete();
    }
    
    this.contexts.delete(executionId);
  }
}
```

**Milestone 3.1**: Execution Engine Core (Week 10)
- ✅ Reactive workflow executor
- ✅ Dependency resolution engine
- ✅ Execution context management
- ✅ Basic error handling and recovery

### 3.2 Advanced Execution Features

#### 3.2.1 Real-time Monitoring and Metrics

```typescript
// Comprehensive execution monitoring
class ExecutionMonitor {
  private metricsCollector: MetricsCollector;
  private healthChecker: HealthChecker;
  private performanceProfiler: PerformanceProfiler;
  
  startMonitoring(context: ExecutionContext): void {
    // Start metrics collection
    this.metricsCollector.startCollection(context.executionId);
    
    // Monitor node performance
    this.monitorNodePerformance(context);
    
    // Monitor stream health
    this.monitorStreamHealth(context);
    
    // Monitor resource usage
    this.monitorResourceUsage(context);
  }
  
  private monitorNodePerformance(context: ExecutionContext): void {
    const executionQuery = context.ecsWorld.query(ExecutionStateComponent);
    
    for (const [entity, executionState] of executionQuery) {
      // Track execution times
      if (executionState.status === 'running') {
        this.performanceProfiler.startTimer(`node-${entity}`);
      } else if (executionState.status === 'completed') {
        const duration = this.performanceProfiler.endTimer(`node-${entity}`);
        this.updateNodeMetrics(entity, duration);
      }
    }
  }
  
  private monitorStreamHealth(context: ExecutionContext): void {
    for (const [streamId, stream] of context.activeStreams) {
      // Monitor stream backpressure
      const backpressure = stream.getBackpressureLevel();
      if (backpressure > 0.8) {
        this.emit('stream-backpressure-warning', { streamId, level: backpressure });
      }
      
      // Monitor error rates
      const errorRate = stream.getErrorRate();
      if (errorRate > 0.1) {
        this.emit('stream-error-rate-warning', { streamId, rate: errorRate });
      }
    }
  }
}
```

#### 3.2.2 Error Recovery Mechanisms

```typescript
// Sophisticated error handling and recovery
class ErrorRecoveryManager {
  private recoveryStrategies: Map<ErrorType, RecoveryStrategy> = new Map();
  
  registerRecoveryStrategy(errorType: ErrorType, strategy: RecoveryStrategy): void {
    this.recoveryStrategies.set(errorType, strategy);
  }
  
  async handleExecutionError(
    context: ExecutionContext,
    error: ExecutionError
  ): Promise<RecoveryResult> {
    const strategy = this.recoveryStrategies.get(error.type);
    
    if (!strategy) {
      return { action: 'fail', reason: 'No recovery strategy found' };
    }
    
    try {
      return await strategy.recover(context, error);
    } catch (recoveryError) {
      return {
        action: 'fail',
        reason: 'Recovery strategy failed',
        originalError: error,
        recoveryError
      };
    }
  }
}

// Built-in recovery strategies
class RetryRecoveryStrategy implements RecoveryStrategy {
  constructor(private config: RetryConfig) {}
  
  async recover(
    context: ExecutionContext,
    error: ExecutionError
  ): Promise<RecoveryResult> {
    if (error.retryCount >= this.config.maxRetries) {
      return { action: 'fail', reason: 'Max retries exceeded' };
    }
    
    const delay = this.calculateBackoffDelay(error.retryCount);
    await this.sleep(delay);
    
    return { action: 'retry', delay };
  }
}

class FallbackRecoveryStrategy implements RecoveryStrategy {
  constructor(private fallbackNode: NodeDefinition) {}
  
  async recover(
    context: ExecutionContext,
    error: ExecutionError
  ): Promise<RecoveryResult> {
    // Replace failed node with fallback
    const fallbackEntity = context.ecsWorld.createEntity([
      // ... fallback node components
    ]);
    
    return { action: 'continue', replacementEntity: fallbackEntity };
  }
}
```

**Milestone 3.2**: Advanced Execution Features (Week 12)
- ✅ Real-time monitoring and metrics
- ✅ Error recovery mechanisms
- ✅ Performance profiling and optimization
- ✅ Execution debugging tools

---

## 4. TypeScript Workflow DSL

### 4.1 Core DSL Design

#### 4.1.1 Workflow Base Class and Decorators

```typescript
// Core workflow DSL foundation
abstract class WorkflowBase {
  protected nodes: Map<string, NodeInstance> = new Map();
  protected connections: ConnectionDefinition[] = [];
  protected config: WorkflowConfig = {};
  
  // Abstract method that users implement
  abstract build(): void;
  
  // DSL methods for workflow construction
  protected addNode<T extends NodeFunction>(
    nodeFunction: T,
    config?: NodeConfig
  ): NodeInstance<T> {
    const nodeId = generateNodeId();
    const instance = new NodeInstance(nodeId, nodeFunction, config);
    
    this.nodes.set(nodeId, instance);
    return instance;
  }
  
  protected connectReactive<T, U>(
    source: SlotReference<T>,
    target: SlotReference<U>,
    config?: ConnectionConfig
  ): ConnectionDefinition {
    const connection: ConnectionDefinition = {
      id: generateConnectionId(),
      from: { nodeId: source.nodeId, slot: source.slotName },
      to: { nodeId: target.nodeId, slot: target.slotName },
      config: config || {}
    };
    
    this.connections.push(connection);
    return connection;
  }
  
  protected fanOut<T>(
    source: SlotReference<T>,
    targets: Array<{
      target: SlotReference<any>;
      filter?: (value: T) => boolean;
      transform?: (value: T) => any;
    }>
  ): ConnectionDefinition[] {
    return targets.map(({ target, filter, transform }) =>
      this.connectReactive(source, target, { filter, transform })
    );
  }
  
  protected fanIn<T>(
    sources: Array<{
      source: SlotReference<T>;
      priority?: number;
      tag?: string;
    }>,
    target: SlotReference<T>,
    config?: FanInConfig
  ): ConnectionDefinition[] {
    // Implementation for fan-in pattern
    return sources.map(({ source, priority, tag }) =>
      this.connectReactive(source, target, { priority, tag, ...config })
    );
  }
  
  // Generate workflow definition for execution
  toDefinition(): WorkflowDefinition {
    // Build the workflow
    this.build();
    
    return {
      id: generateWorkflowId(),
      nodes: Array.from(this.nodes.values()).map(node => node.toDefinition()),
      connections: this.connections,
      config: this.config
    };
  }
}

// Decorators for enhanced DSL experience
function NodeInfo(info: { uuid: string; displayName: string; version?: string }) {
  return function (target: any) {
    target._nodeInfo = info;
  };
}

function Position(position: { x: number; y: number }) {
  return function (target: any) {
    target._position = position;
  };
}

function InputSlots(slots: Record<string, SlotDefinition>) {
  return function (target: any) {
    target._inputSlots = slots;
  };
}

function OutputSlots(slots: Record<string, SlotDefinition>) {
  return function (target: any) {
    target._outputSlots = slots;
  };
}

// Example node function with decorators
@NodeInfo({ uuid: "webhook-reactive", displayName: "Reactive Webhook" })
@Position({ x: 50, y: 100 })
@OutputSlots({
  data: {
    type: 'stream',
    reactive: true,
    displayName: 'Request Stream',
    color: '#2196F3'
  }
})
function reactiveWebhook(
  @OutputSlots() outputs: OutputSlotCollection,
  @Config() config: WebhookConfig
): NodeRuntime {
  const requestStream = createWebhookStream(config.path);
  
  requestStream.subscribe(request => {
    outputs.data.emit({
      body: request.body,
      headers: request.headers,
      timestamp: Date.now()
    });
  });
  
  return {
    dispose: () => requestStream.unsubscribe()
  };
}
```

#### 4.1.2 Type-Safe Slot System

```typescript
// Comprehensive type safety for slot connections
interface SlotReference<T = any> {
  nodeId: string;
  slotName: string;
  nodeInstance: NodeInstance;
  dataType: T;
}

class NodeInstance<T extends NodeFunction = NodeFunction> {
  public readonly slots: NodeSlots<T>;
  
  constructor(
    public readonly id: string,
    private nodeFunction: T,
    private config?: NodeConfig
  ) {
    this.slots = this.createSlotReferences();
  }
  
  private createSlotReferences(): NodeSlots<T> {
    const inputSlots = this.nodeFunction._inputSlots || {};
    const outputSlots = this.nodeFunction._outputSlots || {};
    
    const slots: any = {};
    
    // Create input slot references
    for (const [slotName, slotDef] of Object.entries(inputSlots)) {
      Object.defineProperty(slots, slotName, {
        get: () => new SlotReference({
          nodeId: this.id,
          slotName,
          nodeInstance: this,
          dataType: slotDef.dataType
        })
      });
    }
    
    // Create output slot references
    for (const [slotName, slotDef] of Object.entries(outputSlots)) {
      Object.defineProperty(slots, slotName, {
        get: () => new SlotReference({
          nodeId: this.id,
          slotName,
          nodeInstance: this,
          dataType: slotDef.dataType
        })
      });
    }
    
    return slots;
  }
}

// Type-safe connection validation
type ValidConnection<S, T> = S extends T ? ConnectionDefinition : never;

function validateConnection<S, T>(
  source: SlotReference<S>,
  target: SlotReference<T>
): ValidConnection<S, T> {
  // Runtime type validation
  if (!isCompatibleType(source.dataType, target.dataType)) {
    throw new Error(
      `Incompatible slot types: ${source.dataType} -> ${target.dataType}`
    );
  }
  
  return {
    from: { nodeId: source.nodeId, slot: source.slotName },
    to: { nodeId: target.nodeId, slot: target.slotName }
  } as ValidConnection<S, T>;
}
```

#### 4.1.3 Advanced DSL Features

```typescript
// Advanced workflow patterns and helpers
abstract class AdvancedWorkflowBase extends WorkflowBase {
  
  // Pipeline pattern with type safety
  protected pipeline<T>(stages: PipelineStage<T>[]): PipelineDefinition {
    const connections: ConnectionDefinition[] = [];
    
    for (let i = 0; i < stages.length - 1; i++) {
      const current = stages[i];
      const next = stages[i + 1];
      
      connections.push(
        this.connectReactive(current.output, next.input, {
          transform: current.transform,
          filter: current.filter,
          ...current.config
        })
      );
    }
    
    return { stages, connections };
  }
  
  // Conditional routing
  protected route<T>(
    source: SlotReference<T>,
    routes: Array<{
      condition: (value: T) => boolean;
      target: SlotReference<any>;
      transform?: (value: T) => any;
    }>
  ): ConnectionDefinition[] {
    return routes.map(route =>
      this.connectReactive(source, route.target, {
        filter: route.condition,
        transform: route.transform
      })
    );
  }
  
  // Stream aggregation
  protected aggregateStreams<T extends any[]>(
    sources: SlotReferences<T>,
    target: SlotReference<any>,
    config: AggregationConfig
  ): ConnectionDefinition {
    // Create aggregation node
    const aggregator = this.addNode(streamAggregator, {
      windowSize: config.windowSize,
      aggregationFn: config.aggregationFn
    });
    
    // Connect all sources to aggregator
    const inputConnections = sources.map(source =>
      this.connectReactive(source, aggregator.slots.input)
    );
    
    // Connect aggregator to target
    const outputConnection = this.connectReactive(
      aggregator.slots.output,
      target
    );
    
    return outputConnection;
  }
  
  // Error boundaries
  protected withErrorBoundary<T>(
    operation: () => T,
    errorHandler: ErrorHandlerConfig
  ): T {
    try {
      return operation();
    } catch (error) {
      this.handleWorkflowError(error, errorHandler);
      throw error;
    }
  }
  
  // Workflow composition
  protected includeSubworkflow(
    subworkflow: WorkflowBase,
    mapping: SlotMapping
  ): SubworkflowInstance {
    const subworkflowDef = subworkflow.toDefinition();
    
    // Map external slots to subworkflow slots
    for (const [externalSlot, internalSlot] of Object.entries(mapping.inputs)) {
      this.connectReactive(externalSlot, internalSlot);
    }
    
    for (const [internalSlot, externalSlot] of Object.entries(mapping.outputs)) {
      this.connectReactive(internalSlot, externalSlot);
    }
    
    return new SubworkflowInstance(subworkflowDef, mapping);
  }
}
```

**Milestone 4.1**: Core DSL Implementation (Week 14)
- ✅ Workflow base class and decorators
- ✅ Type-safe slot system
- ✅ Advanced DSL features
- ✅ DSL validation and error reporting

### 4.2 Advanced DSL Features

#### 4.2.1 AI-Assisted Code Generation

```typescript
// AI integration for workflow DSL enhancement
class WorkflowAIAssistant {
  
  generateNodeFromDescription(
    description: string,
    context: WorkflowContext
  ): Promise<NodeDefinition> {
    return this.aiService.generateCode({
      prompt: `Generate a TypeScript workflow node for: ${description}`,
      context: {
        availableSlots: context.availableSlots,
        existingNodes: context.nodes,
        dataSchema: context.schema
      },
      constraints: {
        mustUseDecorators: true,
        mustImplementSlots: true,
        mustHandleErrors: true
      }
    });
  }
  
  suggestConnections(
    sourceSlot: SlotReference,
    availableTargets: SlotReference[]
  ): ConnectionSuggestion[] {
    return this.aiService.analyzeConnections({
      source: sourceSlot,
      targets: availableTargets,
      dataFlow: this.analyzeDataFlow(sourceSlot),
      semantics: this.analyzeSemantics(sourceSlot)
    });
  }
  
  optimizeWorkflow(workflow: WorkflowDefinition): WorkflowOptimization {
    return this.aiService.optimize({
      nodes: workflow.nodes,
      connections: workflow.connections,
      metrics: this.getPerformanceMetrics(workflow),
      constraints: this.getOptimizationConstraints()
    });
  }
}
```

#### 4.2.2 DSL Extensions and Plugins

```typescript
// Extensible DSL architecture
interface DSLExtension {
  name: string;
  version: string;
  decorators?: DecoratorRegistry;
  nodeTypes?: NodeTypeRegistry;
  operators?: OperatorRegistry;
  validators?: ValidatorRegistry;
}

class DSLExtensionManager {
  private extensions: Map<string, DSLExtension> = new Map();
  
  registerExtension(extension: DSLExtension): void {
    this.extensions.set(extension.name, extension);
    
    // Register decorators
    if (extension.decorators) {
      this.registerDecorators(extension.decorators);
    }
    
    // Register node types
    if (extension.nodeTypes) {
      this.registerNodeTypes(extension.nodeTypes);
    }
    
    // Register operators
    if (extension.operators) {
      this.registerOperators(extension.operators);
    }
  }
  
  // Example extension: Database operations
  createDatabaseExtension(): DSLExtension {
    return {
      name: 'database',
      version: '1.0.0',
      nodeTypes: new Map([
        ['database-query', createDatabaseQueryNode],
        ['database-insert', createDatabaseInsertNode],
        ['database-update', createDatabaseUpdateNode]
      ]),
      decorators: new Map([
        ['DatabaseConfig', createDatabaseConfigDecorator],
        ['Transaction', createTransactionDecorator]
      ])
    };
  }
}
```

**Milestone 4.2**: Advanced DSL Features (Week 16)
- ✅ AI-assisted code generation
- ✅ DSL extensions and plugins
- ✅ Advanced workflow patterns
- ✅ DSL performance optimization

---

## 5. Integration and Testing

### 5.1 System Integration

#### 5.1.1 Component Integration Testing

```typescript
// Comprehensive integration testing framework
class SystemIntegrationTests {
  
  @Test('ECS and Reactive Streams Integration')
  async testECSStreamIntegration(): Promise<void> {
    const world = new ECSWorld();
    const streamManager = new SlotStreamManager();
    
    // Create test entities with reactive slots
    const sourceEntity = world.createEntity([
      new NodeInfoComponent({ uuid: 'source', displayName: 'Source' }),
      new SlotMetadataComponent({
        inputs: new Map(),
        outputs: new Map([['data', { type: 'stream', reactive: true }]])
      })
    ]);
    
    const targetEntity = world.createEntity([
      new NodeInfoComponent({ uuid: 'target', displayName: 'Target' }),
      new SlotMetadataComponent({
        inputs: new Map([['input', { type: 'stream', reactive: true }]]),
        outputs: new Map()
      })
    ]);
    
    // Test stream connection
    const sourceStream = streamManager.createSlotStream(`${sourceEntity}:data`);
    const targetStream = streamManager.createSlotStream(`${targetEntity}:input`);
    
    streamManager.connectSlots(
      `${sourceEntity}:data`,
      `${targetEntity}:input`,
      { transform: (x) => x * 2 }
    );
    
    // Verify data flow
    let receivedValue: any;
    targetStream.subscribe({ next: (value) => receivedValue = value });
    
    sourceStream.emit(42);
    
    await new Promise(resolve => setTimeout(resolve, 10));
    assert.equal(receivedValue, 84);
  }
  
  @Test('Workflow Execution Integration')
  async testWorkflowExecution(): Promise<void> {
    // Create test workflow using DSL
    class TestWorkflow extends WorkflowBase {
      build() {
        const source = this.addNode(testSourceNode);
        const processor = this.addNode(testProcessorNode);
        const sink = this.addNode(testSinkNode);
        
        this.connectReactive(source.slots.output, processor.slots.input);
        this.connectReactive(processor.slots.output, sink.slots.input);
      }
    }
    
    const workflow = new TestWorkflow();
    const executor = new ReactiveWorkflowExecutor();
    
    const result = await executor.executeWorkflow(workflow.toDefinition());
    
    assert.equal(result.status, 'completed');
    assert.isTrue(result.metrics.totalExecutionTime > 0);
  }
}
```

#### 5.1.2 Performance Benchmarks

```typescript
// Performance benchmarking suite
class PerformanceBenchmarks {
  
  @Benchmark('ECS Entity Creation')
  async benchmarkEntityCreation(): Promise<BenchmarkResult> {
    const world = new ECSWorld();
    const iterations = 100000;
    
    const startTime = performance.now();
    
    for (let i = 0; i < iterations; i++) {
      world.createEntity([
        new NodeInfoComponent({ uuid: `node-${i}`, displayName: `Node ${i}` }),
        new PositionComponent({ x: i % 100, y: Math.floor(i / 100) }),
        new ExecutionStateComponent({ status: 'idle', metrics: {} })
      ]);
    }
    
    const endTime = performance.now();
    const duration = endTime - startTime;
    
    return {
      operation: 'ECS Entity Creation',
      iterations,
      totalTime: duration,
      operationsPerSecond: iterations / (duration / 1000),
      memoryUsage: process.memoryUsage()
    };
  }
  
  @Benchmark('Stream Throughput')
  async benchmarkStreamThroughput(): Promise<BenchmarkResult> {
    const stream = new ReactiveStreamImpl<number>(
      new TestStreamSource(),
      new TestScheduler()
    );
    
    const iterations = 1000000;
    let receivedCount = 0;
    
    stream.subscribe({
      next: () => receivedCount++,
      error: () => {},
      complete: () => {}
    });
    
    const startTime = performance.now();
    
    for (let i = 0; i < iterations; i++) {
      stream.emit(i);
    }
    
    // Wait for all emissions to be processed
    await new Promise(resolve => {
      const checkComplete = () => {
        if (receivedCount === iterations) {
          resolve(void 0);
        } else {
          setTimeout(checkComplete, 1);
        }
      };
      checkComplete();
    });
    
    const endTime = performance.now();
    const duration = endTime - startTime;
    
    return {
      operation: 'Stream Throughput',
      iterations,
      totalTime: duration,
      operationsPerSecond: iterations / (duration / 1000),
      memoryUsage: process.memoryUsage()
    };
  }
}
```

### 5.2 Documentation and Examples

#### 5.2.1 API Documentation Generation

```typescript
// Automated documentation generation from DSL
class DSLDocumentationGenerator {
  
  generateNodeDocumentation(nodeFunction: NodeFunction): NodeDocumentation {
    const nodeInfo = nodeFunction._nodeInfo;
    const inputSlots = nodeFunction._inputSlots || {};
    const outputSlots = nodeFunction._outputSlots || {};
    
    return {
      uuid: nodeInfo.uuid,
      displayName: nodeInfo.displayName,
      description: this.extractDescription(nodeFunction),
      version: nodeInfo.version || '1.0.0',
      
      inputSlots: Object.entries(inputSlots).map(([name, slot]) => ({
        name,
        type: slot.type,
        reactive: slot.reactive,
        description: slot.description,
        required: slot.required,
        schema: this.generateSchema(slot.dataType)
      })),
      
      outputSlots: Object.entries(outputSlots).map(([name, slot]) => ({
        name,
        type: slot.type,
        reactive: slot.reactive,
        description: slot.description,
        schema: this.generateSchema(slot.dataType)
      })),
      
      examples: this.generateExamples(nodeFunction),
      codeSnippet: this.generateCodeSnippet(nodeFunction)
    };
  }
  
  generateWorkflowDocumentation(
    workflowClass: typeof WorkflowBase
  ): WorkflowDocumentation {
    const instance = new workflowClass();
    const definition = instance.toDefinition();
    
    return {
      name: workflowClass.name,
      description: this.extractDescription(workflowClass),
      nodes: definition.nodes.map(node => 
        this.generateNodeDocumentation(node.nodeFunction)
      ),
      connections: definition.connections.map(conn => ({
        from: `${conn.from.nodeId}.${conn.from.slot}`,
        to: `${conn.to.nodeId}.${conn.to.slot}`,
        description: this.describeConnection(conn)
      })),
      dataFlow: this.analyzeDataFlow(definition),
      examples: this.generateWorkflowExamples(workflowClass)
    };
  }
}
```

#### 5.2.2 Example Workflows

```typescript
// Comprehensive example workflows for testing and documentation
class ExampleWorkflows {
  
  // Simple data processing pipeline
  static createSimpleDataPipeline(): WorkflowBase {
    return new class extends WorkflowBase {
      build() {
        const source = this.addNode(dataSourceNode, {
          config: { 
            url: 'https://api.example.com/data',
            interval: 5000 
          }
        });
        
        const validator = this.addNode(dataValidatorNode, {
          config: {
            schema: {
              type: 'object',
              properties: {
                id: { type: 'string' },
                value: { type: 'number' }
              }
            }
          }
        });
        
        const processor = this.addNode(dataProcessorNode, {
          config: {
            operations: ['normalize', 'enrich']
          }
        });
        
        const sink = this.addNode(dataSinkNode, {
          config: {
            destination: 'database',
            table: 'processed_data'
          }
        });
        
        // Connect with error handling
        this.connectReactive(source.slots.data, validator.slots.input, {
          onError: (error, data) => {
            console.error('Data source error:', error);
            return null; // Skip invalid data
          }
        });
        
        this.connectReactive(validator.slots.valid, processor.slots.input);
        this.connectReactive(processor.slots.output, sink.slots.input);
        
        // Handle validation failures
        this.connectReactive(validator.slots.invalid, sink.slots.input, {
          transform: (data) => ({ ...data, validation_error: true })
        });
      }
    };
  }
  
  // Complex real-time analytics workflow
  static createAnalyticsWorkflow(): WorkflowBase {
    return new class extends WorkflowBase {
      build() {
        // Data sources
        const webEvents = this.addNode(webEventSourceNode);
        const mobileEvents = this.addNode(mobileEventSourceNode);
        const apiEvents = this.addNode(apiEventSourceNode);
        
        // Processing pipeline
        const eventEnricher = this.addNode(eventEnricherNode);
        const sessionizer = this.addNode(sessionizerNode);
        const aggregator = this.addNode(realTimeAggregatorNode);
        const anomalyDetector = this.addNode(anomalyDetectorNode);
        
        // Output destinations
        const dashboard = this.addNode(dashboardUpdaterNode);
        const alerter = this.addNode(alertingSystemNode);
        const warehouse = this.addNode(dataWarehouseNode);
        
        // Fan-in from multiple sources
        this.fanIn([
          { source: webEvents.slots.events, tag: 'web', priority: 1 },
          { source: mobileEvents.slots.events, tag: 'mobile', priority: 1 },
          { source: apiEvents.slots.events, tag: 'api', priority: 2 }
        ], eventEnricher.slots.input, {
          mergeStrategy: 'timestamp',
          bufferSize: 1000
        });
        
        // Processing pipeline
        this.pipeline([
          {
            from: eventEnricher.slots.enriched,
            to: sessionizer.slots.input,
            transform: (event) => ({
              ...event,
              user_segment: getUserSegment(event.user_id)
            })
          },
          {
            from: sessionizer.slots.sessions,
            to: aggregator.slots.input,
            window: {
              type: 'session',
              sessionTimeout: 30 * 60 * 1000
            }
          },
          {
            from: aggregator.slots.metrics,
            to: anomalyDetector.slots.input
          }
        ]);
        
        // Fan-out to multiple destinations
        this.fanOut(aggregator.slots.metrics, [
          { target: dashboard.slots.update },
          { target: warehouse.slots.input, transform: addBatchMetadata }
        ]);
        
        // Alert on anomalies
        this.connectReactive(anomalyDetector.slots.anomalies, alerter.slots.input, {
          filter: (anomaly) => anomaly.severity > 0.8
        });
      }
    };
  }
}
```

**Milestone 5.1**: Integration and Testing (Week 16)
- ✅ System integration testing
- ✅ Performance benchmarking
- ✅ Documentation generation
- ✅ Example workflows

---

## 6. Success Criteria and Deliverables

### 6.1 Technical Success Criteria

#### Performance Requirements
- **ECS Performance**: Handle >100,000 entities with <16ms frame time
- **Stream Throughput**: Process >1M events/second with <10ms latency
- **Memory Efficiency**: <1GB memory usage for complex workflows
- **Startup Time**: Workflow initialization <100ms

#### Functional Requirements
- **Type Safety**: 100% TypeScript type coverage
- **Error Handling**: Graceful handling of all error conditions
- **Monitoring**: Real-time metrics and health monitoring
- **Extensibility**: Plugin architecture for custom nodes

### 6.2 Deliverables

#### Core Libraries
1. **@n9n/ecs-core**: Entity Component System implementation
2. **@n9n/reactive-streams**: Reactive stream library
3. **@n9n/execution-engine**: Workflow execution engine  
4. **@n9n/workflow-dsl**: TypeScript DSL framework

#### Documentation
1. **Architecture Guide**: Complete system architecture documentation
2. **API Reference**: Comprehensive API documentation
3. **Tutorial Series**: Step-by-step workflow creation guides
4. **Examples Repository**: Collection of example workflows

#### Development Tools
1. **DSL Validator**: Static analysis for workflow validation
2. **Performance Profiler**: Execution performance analysis
3. **Debugging Tools**: Workflow debugging and inspection
4. **Code Generator**: Template generation for new nodes

### 6.3 Phase 1 Acceptance Criteria

#### Must Have
- ✅ Complete ECS foundation with component system
- ✅ Functional reactive stream implementation
- ✅ Basic workflow execution engine
- ✅ TypeScript DSL with decorators
- ✅ Type-safe slot connections
- ✅ Error handling and recovery
- ✅ Performance benchmarks
- ✅ Integration tests

#### Should Have
- ✅ Real-time monitoring and metrics
- ✅ Advanced stream operators
- ✅ Workflow debugging tools
- ✅ API documentation generation
- ✅ Example workflows

#### Could Have
- ⏳ AI-assisted code generation
- ⏳ Visual workflow debugging
- ⏳ Advanced optimization features
- ⏳ Plugin marketplace integration

---

## 7. Risk Assessment and Mitigation

### 7.1 Technical Risks

#### High-Performance ECS Implementation
**Risk**: ECS performance doesn't meet requirements  
**Mitigation**: 
- Early performance benchmarking
- Iterative optimization approach
- Fallback to simpler architecture if needed

#### Reactive Stream Complexity
**Risk**: Stream backpressure and error handling complexity  
**Mitigation**:
- Start with simple operators, add complexity incrementally
- Extensive testing with various load patterns
- Reference existing libraries (RxJS, etc.)

#### TypeScript DSL Usability
**Risk**: DSL too complex for developers  
**Mitigation**:
- User testing with target developers
- Iterative API design based on feedback
- Comprehensive documentation and examples

### 7.2 Project Risks

#### Timeline Constraints
**Risk**: 16-week timeline may be aggressive  
**Mitigation**:
- Prioritize core features over nice-to-haves
- Parallel development where possible
- Weekly milestone reviews

#### Team Coordination
**Risk**: 4-6 engineers need tight coordination  
**Mitigation**:
- Clear component boundaries
- Daily standups and weekly technical reviews
- Shared documentation and code standards

---

## 8. Next Steps

### Immediate Actions (Week 1)
1. **Team Formation**: Assemble 4-6 engineer team
2. **Environment Setup**: Development infrastructure and tooling
3. **Architecture Review**: Final architecture validation
4. **Project Planning**: Detailed sprint planning for 16 weeks

### Week 2-4: Foundation
1. **ECS Core Implementation**: Entity, Component, System architecture
2. **Basic Stream Implementation**: Core reactive stream functionality
3. **Initial DSL Design**: Decorator and base class implementation

### Week 5-8: Core Features
1. **Advanced ECS Features**: Archetype optimization, queries
2. **Stream Operators**: Transform, filter, buffer, retry operators
3. **Execution Engine**: Basic workflow execution and dependency resolution

### Week 9-12: Integration
1. **Stream-ECS Integration**: Connect reactive streams with ECS
2. **Error Handling**: Comprehensive error recovery mechanisms
3. **Performance Optimization**: Benchmarking and optimization

### Week 13-16: Polish and Testing
1. **Advanced DSL Features**: AI assistance, extensions
2. **Documentation**: API docs, tutorials, examples
3. **Final Testing**: Integration tests, performance validation

---

This Phase 1 plan establishes the foundational architecture that will enable n9n to deliver on its vision of code-first, reactive workflow automation. The combination of ECS architecture with reactive streams provides a unique foundation for high-performance, scalable workflow execution while maintaining the developer experience that makes n9n distinctive in the market.

The success of Phase 1 will enable subsequent phases to build the visual editor, AI integrations, and enterprise features that complete the n9n platform vision.
