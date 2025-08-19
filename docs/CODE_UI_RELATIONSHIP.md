# Code <-> UI Relationship Design

## Core Philosophy: Code as Source of Truth

The n9n platform operates on the principle that **code is the ultimate source of truth**. The visual editor is a sophisticated view and manipulation layer that generates, modifies, and reflects the underlying code. This approach ensures:

- **Version Control**: Full Git integration for workflows
- **Type Safety**: Complete TypeScript support with compile-time validation
- **Extensibility**: Developers can extend beyond UI limitations
- **Reproducibility**: Workflows are deterministic and testable

## 1. Abstraction Layer Architecture

### 1.1 Three-Tier Abstraction Model

```typescript
// Layer 1: Code Layer (Source of Truth)
class ReactiveWorkflow extends WorkflowBase {
  public build() {
    const webhook = this.addNode(reactiveWebhook, { 
      config: { path: '/api/contact' } 
    });
    
    this.connectReactive(webhook.slots.data, processor.slots.input, {
      transform: (data) => ({ ...data, enriched: true }),
      filter: (data) => data.email && isValidEmail(data.email),
      backpressure: { strategy: 'buffer', bufferSize: 1000 }
    });
  }
}

// Layer 2: AST Representation (Intermediate)
interface WorkflowAST {
  nodes: NodeDefinition[];
  connections: ConnectionDefinition[];
  metadata: WorkflowMetadata;
}

// Layer 3: Visual Representation (UI)
interface VisualWorkflow {
  canvas: CanvasDefinition;
  visualElements: UIElement[];
  interactionHandlers: InteractionHandler[];
}
```

### 1.2 Abstraction Boundaries

| **Aspect** | **Visual Editor** | **Code Editor** | **Hybrid Mode** |
|------------|------------------|-----------------|-----------------|
| **Node Placement** | ✅ Drag & Drop | ❌ Auto-layout | ✅ Visual + Code |
| **Basic Connections** | ✅ Click & Connect | ❌ Method calls | ✅ Both |
| **Simple Transforms** | ✅ Form Fields | ❌ Code required | ✅ Form with code fallback |
| **Complex Logic** | ❌ Code required | ✅ Full TypeScript | ✅ Inline code editor |
| **Type Definitions** | ❌ Code required | ✅ Full TypeScript | ❌ View-only |
| **Custom Nodes** | ❌ Code required | ✅ Full control | ❌ Import from code |

## 2. Bidirectional Synchronization Mechanisms

### 2.1 Change Detection and Propagation

```typescript
class WorkflowSynchronizer {
  private codeParser: TypeScriptParser;
  private astManager: ASTManager;
  private uiRenderer: UIRenderer;
  private changeDetector: ChangeDetector;

  constructor() {
    this.setupBidirectionalSync();
  }

  private setupBidirectionalSync() {
    // Code -> UI sync
    this.changeDetector.onCodeChange((changes: CodeChange[]) => {
      const updatedAST = this.codeParser.parseWorkflow(changes.sourceCode);
      const uiUpdates = this.astManager.diffForUI(updatedAST);
      this.uiRenderer.applyUpdates(uiUpdates);
    });

    // UI -> Code sync  
    this.changeDetector.onUIChange((changes: UIChange[]) => {
      const codeUpdates = this.astManager.diffForCode(changes);
      const updatedCode = this.codeParser.generateCode(codeUpdates);
      this.applyCodeChanges(updatedCode);
    });
  }

  // Conflict resolution when both code and UI change simultaneously
  private resolveConflicts(codeChanges: CodeChange[], uiChanges: UIChange[]): Resolution {
    // Code changes take precedence for complex logic
    // UI changes take precedence for layout and simple configurations
    return this.conflictResolver.resolve(codeChanges, uiChanges);
  }
}
```

### 2.2 Change Categorization

```typescript
enum ChangeType {
  // UI-Safe Changes (can be made in visual editor)
  NODE_POSITION = 'node_position',
  NODE_CONFIG = 'node_config', 
  BASIC_CONNECTION = 'basic_connection',
  SLOT_CONFIGURATION = 'slot_configuration',
  
  // Code-Only Changes (require code editor)
  CUSTOM_TRANSFORM = 'custom_transform',
  COMPLEX_FILTER = 'complex_filter', 
  TYPE_DEFINITION = 'type_definition',
  CUSTOM_NODE = 'custom_node',
  
  // Hybrid Changes (can be started in UI, completed in code)
  CONNECTION_WITH_LOGIC = 'connection_with_logic',
  CONDITIONAL_ROUTING = 'conditional_routing'
}

interface ChangeHandler {
  canHandleInUI(change: Change): boolean;
  requiresCodeEditor(change: Change): boolean;
  generateUIPreview(change: Change): UIPreview;
  generateCodeSnippet(change: Change): string;
}
```

## 3. Visual Representations for Complex Code Logic

### 3.1 Transform Function Visualization

```typescript
// Complex transform in code
this.connectReactive(source.slots.data, target.slots.input, {
  transform: (data) => {
    const enriched = enrichUserData(data);
    const validated = validateBusinessRules(enriched);
    return {
      ...validated,
      priority: calculatePriority(validated),
      routing: determineRouting(validated.type),
      timestamp: Date.now()
    };
  }
});

// Visual representation in UI
interface TransformVisualizer {
  type: 'complex_transform';
  preview: {
    inputSchema: JSONSchema;
    outputSchema: JSONSchema;
    codePreview: string; // First few lines
    hasFullCode: boolean;
  };
  editMode: {
    inlineEditor: boolean;
    fullCodeEditor: boolean;
    assistedBuilder: boolean; // For simple transforms
  };
}
```

### 3.2 Visual Transform Builder for Simple Cases

```typescript
// Simple transforms can be built visually
interface VisualTransformBuilder {
  operations: TransformOperation[];
}

interface TransformOperation {
  type: 'pick' | 'rename' | 'compute' | 'filter' | 'enrich';
  config: {
    // Pick operation
    pick?: { fields: string[] };
    
    // Rename operation  
    rename?: { from: string; to: string };
    
    // Compute operation (with formula builder)
    compute?: { 
      field: string; 
      formula: Formula; // Visual formula builder
      dependencies: string[];
    };
    
    // Filter operation (with condition builder)
    filter?: {
      conditions: VisualCondition[];
      logic: 'AND' | 'OR';
    };
  };
}

// Generated code from visual builder
function generateTransformCode(operations: TransformOperation[]): string {
  return `(data) => {
    ${operations.map(op => generateOperationCode(op)).join('\n  ')}
    return result;
  }`;
}
```

### 3.3 Connection Configuration UI

```typescript
interface ConnectionConfigUI {
  // Basic configuration (always in UI)
  basic: {
    sourceSlot: SlotReference;
    targetSlot: SlotReference;
    enabled: boolean;
  };
  
  // Transform configuration
  transform: {
    mode: 'none' | 'simple' | 'complex';
    simple?: VisualTransformBuilder;
    complex?: {
      code: string;
      language: 'typescript' | 'javascript';
      preview: TransformPreview;
    };
  };
  
  // Filter configuration
  filter: {
    mode: 'none' | 'visual' | 'code';
    visual?: VisualConditionBuilder;
    code?: {
      expression: string;
      preview: FilterPreview;
    };
  };
  
  // Advanced options
  advanced: {
    backpressure: BackpressureConfig;
    circuitBreaker: CircuitBreakerConfig;
    rateLimit: RateLimitConfig;
    errorHandling: ErrorHandlingConfig;
  };
}
```

## 4. UI Components for Different Editing Modes

### 4.1 Mode-Aware Editor Components

```typescript
// Main workflow editor with mode switching
interface WorkflowEditor {
  currentMode: EditingMode;
  splitView: boolean; // Show code and visual simultaneously
  
  modes: {
    visual: VisualEditor;
    code: CodeEditor;
    hybrid: HybridEditor;
  };
}

enum EditingMode {
  VISUAL_ONLY = 'visual',      // Pure drag-drop, no code
  CODE_ONLY = 'code',          // Pure TypeScript editor
  HYBRID = 'hybrid',           // Visual + inline code editors
  SPLIT_VIEW = 'split'         // Side-by-side code and visual
}

// Visual editor component
interface VisualEditor {
  canvas: WorkflowCanvas;
  palette: NodePalette;
  properties: PropertyPanel;
  connectionManager: VisualConnectionManager;
  
  // Intelligent fallbacks when visual editing hits limits
  fallbackHandlers: {
    openCodeEditor(context: EditingContext): void;
    showCodePreview(element: WorkflowElement): void;
    suggestCodeCompletion(partial: PartialEdit): void;
  };
}

// Code editor component  
interface CodeEditor {
  monaco: MonacoEditor;
  typeChecker: TypeScriptChecker;
  livePreview: VisualPreview;
  
  // Integration with visual editor
  visualHelpers: {
    showVisualPreview(): void;
    highlightVisualElement(codeRange: Range): void;
    insertNodeTemplate(nodeType: string): void;
  };
}

// Hybrid editor for seamless transitions
interface HybridEditor {
  visualEditor: VisualEditor;
  inlineCodeEditors: Map<ElementId, InlineCodeEditor>;
  contextualHelpers: ContextualHelper[];
  
  // Smart editing assistance
  aiAssistant: {
    generateCodeFromDescription(description: string): string;
    explainCode(code: string): string;
    suggestRefactoring(code: string): Refactoring[];
  };
}
```

### 4.2 Property Panel Adaptations

```typescript
interface SmartPropertyPanel {
  adaptToElement(element: WorkflowElement): PropertyPanelConfig;
}

interface PropertyPanelConfig {
  sections: PropertySection[];
  editingHints: EditingHint[];
  codePreview?: CodePreviewConfig;
}

interface PropertySection {
  title: string;
  properties: Property[];
  collapsible: boolean;
  advanced?: boolean;
}

interface Property {
  key: string;
  type: PropertyType;
  editor: PropertyEditor;
  validation: PropertyValidation;
  
  // Connection to code
  codeMapping: {
    path: string; // Path in AST
    transformer?: (value: any) => string; // UI value to code
    parser?: (code: string) => any; // Code to UI value
  };
}

enum PropertyType {
  STRING = 'string',
  NUMBER = 'number', 
  BOOLEAN = 'boolean',
  SELECT = 'select',
  MULTISELECT = 'multiselect',
  OBJECT = 'object',
  ARRAY = 'array',
  CODE = 'code',           // Requires code editor
  FORMULA = 'formula',     // Visual formula builder
  CONDITION = 'condition', // Visual condition builder
  TRANSFORM = 'transform'  // Visual transform builder
}

// Example: Connection property panel
const connectionPropertyConfig: PropertyPanelConfig = {
  sections: [
    {
      title: 'Basic Configuration',
      properties: [
        {
          key: 'enabled',
          type: PropertyType.BOOLEAN,
          editor: { type: 'toggle' },
          codeMapping: { path: 'connection.enabled' }
        }
      ]
    },
    {
      title: 'Transform',
      properties: [
        {
          key: 'transform',
          type: PropertyType.TRANSFORM,
          editor: { 
            type: 'smart_transform',
            modes: ['none', 'visual', 'code'],
            visualBuilder: VisualTransformBuilder,
            codeEditor: MonacoEditor
          },
          codeMapping: {
            path: 'connection.config.transform',
            transformer: (config) => generateTransformCode(config),
            parser: (code) => parseTransformCode(code)
          }
        }
      ]
    }
  ]
};
```

## 5. Code Generation and Parsing

### 5.1 Intelligent Code Generation

```typescript
class WorkflowCodeGenerator {
  generateFromVisual(visualWorkflow: VisualWorkflow): string {
    const ast = this.visualToAST(visualWorkflow);
    return this.astToCode(ast);
  }

  private astToCode(ast: WorkflowAST): string {
    return `
class ${ast.metadata.className} extends WorkflowBase {
  public build() {
    ${this.generateNodeDefinitions(ast.nodes)}
    
    ${this.generateConnections(ast.connections)}
  }
  
  ${this.generateHelperMethods(ast)}
}`;
  }

  private generateConnections(connections: ConnectionDefinition[]): string {
    return connections.map(conn => {
      if (conn.hasComplexLogic) {
        return this.generateComplexConnection(conn);
      } else {
        return this.generateSimpleConnection(conn);
      }
    }).join('\n\n    ');
  }

  private generateComplexConnection(conn: ConnectionDefinition): string {
    return `// Complex connection with custom logic
this.connectReactive(${conn.source}, ${conn.target}, {
  ${this.generateTransform(conn.transform)}
  ${this.generateFilter(conn.filter)}
  ${this.generateAdvancedConfig(conn.advanced)}
});`;
  }

  private generateTransform(transform?: TransformConfig): string {
    if (!transform) return '';
    
    if (transform.type === 'visual') {
      return `transform: ${this.generateTransformFromVisual(transform.config)},`;
    } else {
      return `transform: ${transform.code},`;
    }
  }
}
```

### 5.2 Code Parsing and AST Management

```typescript
class WorkflowCodeParser {
  parseWorkflow(code: string): WorkflowAST {
    const sourceFile = this.typescript.createSourceFile(
      'workflow.ts', 
      code, 
      ScriptTarget.Latest, 
      true
    );
    
    return this.visitWorkflowClass(sourceFile);
  }

  private visitWorkflowClass(node: ClassDeclaration): WorkflowAST {
    const buildMethod = this.findBuildMethod(node);
    const ast: WorkflowAST = {
      nodes: this.extractNodes(buildMethod),
      connections: this.extractConnections(buildMethod),
      metadata: this.extractMetadata(node)
    };
    
    return ast;
  }

  private extractConnections(buildMethod: MethodDeclaration): ConnectionDefinition[] {
    const connections: ConnectionDefinition[] = [];
    
    this.forEachChild(buildMethod, (node) => {
      if (this.isConnectReactiveCall(node)) {
        connections.push(this.parseConnectionCall(node as CallExpression));
      }
    });
    
    return connections;
  }

  private parseConnectionCall(call: CallExpression): ConnectionDefinition {
    const [source, target, config] = call.arguments;
    
    return {
      source: this.parseSlotReference(source),
      target: this.parseSlotReference(target),
      config: config ? this.parseConnectionConfig(config) : undefined,
      hasComplexLogic: this.hasComplexLogic(config)
    };
  }

  private hasComplexLogic(config?: Expression): boolean {
    if (!config) return false;
    
    // Check if transform or filter contains function expressions
    // that are too complex for visual representation
    return this.containsComplexFunction(config);
  }
}
```

## 6. Real-time Synchronization Examples

### 6.1 UI Change -> Code Update

```typescript
// User drags a node in the visual editor
class UIChangeHandler {
  onNodePositionChange(nodeId: string, newPosition: Position) {
    // 1. Update AST
    const ast = this.astManager.getWorkflowAST();
    const node = ast.nodes.find(n => n.id === nodeId);
    node.positionComponent = newPosition;
    
    // 2. Generate code update
    const codeUpdate = this.codeGenerator.generatePositionUpdate(nodeId, newPosition);
    
    // 3. Apply to code editor (if open)
    if (this.codeEditor.isVisible()) {
      this.codeEditor.applyUpdate(codeUpdate);
    }
    
    // 4. Update source file
    this.workflowManager.updateSourceCode(codeUpdate);
  }

  onConnectionCreate(sourceSlot: SlotRef, targetSlot: SlotRef) {
    // 1. Check if connection is possible
    const validation = this.typeChecker.validateConnection(sourceSlot, targetSlot);
    if (!validation.valid) {
      this.showConnectionError(validation.errors);
      return;
    }

    // 2. Create basic connection
    const connectionCode = `this.connectReactive(${sourceSlot.reference}, ${targetSlot.reference});`;
    
    // 3. Insert into build method
    const insertPosition = this.findConnectionInsertPosition();
    this.codeEditor.insertCode(connectionCode, insertPosition);
    
    // 4. Open connection configuration panel
    this.propertyPanel.showConnectionConfig(sourceSlot, targetSlot);
  }
}
```

### 6.2 Code Change -> UI Update

```typescript
// User modifies code in the code editor
class CodeChangeHandler {
  onCodeChange(changes: CodeChange[]) {
    // 1. Parse updated code
    const newAST = this.codeParser.parseWorkflow(changes.newCode);
    const oldAST = this.astManager.getCurrentAST();
    
    // 2. Diff the ASTs
    const diff = this.astDiffer.diff(oldAST, newAST);
    
    // 3. Apply changes to visual editor
    diff.forEach(change => {
      switch (change.type) {
        case 'node_added':
          this.visualEditor.addNode(change.node);
          break;
        case 'node_removed':
          this.visualEditor.removeNode(change.nodeId);
          break;
        case 'connection_modified':
          this.updateConnectionVisual(change.connection);
          break;
        case 'complex_logic_added':
          this.showComplexLogicIndicator(change.element);
          break;
      }
    });
    
    // 4. Update AST manager
    this.astManager.updateAST(newAST);
  }

  private updateConnectionVisual(connection: ConnectionDefinition) {
    const visualConnection = this.visualEditor.getConnection(connection.id);
    
    // Update visual indicators based on connection complexity
    if (connection.hasComplexLogic) {
      visualConnection.addComplexityIndicator();
      visualConnection.showCodePreview(connection.transform?.code);
    } else {
      visualConnection.removeComplexityIndicator();
      visualConnection.updateSimpleConfig(connection.config);
    }
  }
}
```

## 7. Implementation Roadmap

### Phase 1: Foundation
1. ✅ Core AST representation
2. ✅ Basic code parser and generator
3. ✅ Simple visual editor

### Phase 2: Synchronization
1. 🔄 Bidirectional sync engine
2. 🔄 Change detection and diffing
3. 🔄 Conflict resolution

### Phase 3: Advanced Features
1. ⏳ Visual transform builder
2. ⏳ Inline code editors
3. ⏳ AI-assisted code generation

### Phase 4: Polish
1. ⏳ Real-time collaboration
2. ⏳ Advanced debugging tools
3. ⏳ Performance optimization

This design ensures that n9n can seamlessly bridge the gap between visual and code-based workflow development while maintaining code as the ultimate source of truth.
