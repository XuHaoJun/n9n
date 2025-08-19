# UI Component Examples for Code <-> UI Relationship

## 1. Connection Configuration Panel

### Visual Transform Builder

```tsx
interface ConnectionConfigPanel {
  connection: ConnectionDefinition;
  onUpdate: (config: ConnectionConfig) => void;
}

function ConnectionConfigPanel({ connection, onUpdate }: ConnectionConfigPanel) {
  const [transformMode, setTransformMode] = useState<'none' | 'visual' | 'code'>('none');
  const [visualConfig, setVisualConfig] = useState<VisualTransformConfig>();
  const [codeConfig, setCodeConfig] = useState<string>();

  return (
    <Panel title="Connection Configuration">
      
      {/* Basic Settings */}
      <Section title="Basic">
        <Toggle 
          label="Enabled" 
          value={connection.enabled}
          onChange={(enabled) => onUpdate({ ...connection, enabled })}
        />
      </Section>

      {/* Transform Configuration */}
      <Section title="Transform">
        <RadioGroup 
          value={transformMode} 
          onChange={setTransformMode}
          options={[
            { value: 'none', label: 'No Transform' },
            { value: 'visual', label: 'Visual Builder' },
            { value: 'code', label: 'Custom Code' }
          ]}
        />

        {transformMode === 'visual' && (
          <VisualTransformBuilder
            inputSchema={connection.sourceSlot.schema}
            outputSchema={connection.targetSlot.schema}
            config={visualConfig}
            onChange={setVisualConfig}
            onGenerateCode={(code) => {
              setCodeConfig(code);
              onUpdate({ 
                ...connection, 
                transform: { type: 'code', code } 
              });
            }}
          />
        )}

        {transformMode === 'code' && (
          <CodeEditor
            language="typescript"
            value={codeConfig}
            onChange={setCodeConfig}
            schema={{
              input: connection.sourceSlot.schema,
              output: connection.targetSlot.schema
            }}
            onValidate={(isValid) => {
              if (isValid) {
                onUpdate({ 
                  ...connection, 
                  transform: { type: 'code', code: codeConfig } 
                });
              }
            }}
          />
        )}
      </Section>

      {/* Advanced Settings */}
      <CollapsibleSection title="Advanced" defaultCollapsed>
        <BackpressureConfig 
          config={connection.backpressure}
          onChange={(backpressure) => onUpdate({ ...connection, backpressure })}
        />
        <CircuitBreakerConfig 
          config={connection.circuitBreaker}
          onChange={(circuitBreaker) => onUpdate({ ...connection, circuitBreaker })}
        />
      </CollapsibleSection>

      {/* Code Preview */}
      {(transformMode === 'visual' && visualConfig) && (
        <Section title="Generated Code">
          <CodePreview 
            code={generateTransformCode(visualConfig)}
            language="typescript"
            readonly
            copyable
          />
        </Section>
      )}

    </Panel>
  );
}
```

### Visual Transform Builder Component

```tsx
interface VisualTransformBuilder {
  inputSchema: JSONSchema;
  outputSchema: JSONSchema;
  config?: VisualTransformConfig;
  onChange: (config: VisualTransformConfig) => void;
  onGenerateCode: (code: string) => void;
}

function VisualTransformBuilder({ 
  inputSchema, 
  outputSchema, 
  config, 
  onChange, 
  onGenerateCode 
}: VisualTransformBuilder) {
  const [operations, setOperations] = useState<TransformOperation[]>(config?.operations || []);

  const addOperation = (type: OperationType) => {
    const newOp: TransformOperation = {
      id: generateId(),
      type,
      config: getDefaultConfig(type)
    };
    const newOperations = [...operations, newOp];
    setOperations(newOperations);
    updateConfig(newOperations);
  };

  const updateConfig = (ops: TransformOperation[]) => {
    const newConfig = { operations: ops };
    onChange(newConfig);
    onGenerateCode(generateCodeFromOperations(ops));
  };

  return (
    <div className="visual-transform-builder">
      
      {/* Input Schema Display */}
      <div className="schema-panel">
        <h4>Input Data</h4>
        <SchemaViewer schema={inputSchema} />
      </div>

      {/* Operations Pipeline */}
      <div className="operations-pipeline">
        <h4>Transform Operations</h4>
        
        {operations.map((operation, index) => (
          <OperationCard 
            key={operation.id}
            operation={operation}
            index={index}
            inputSchema={getSchemaAtStep(index)}
            onUpdate={(updated) => {
              const newOps = [...operations];
              newOps[index] = updated;
              setOperations(newOps);
              updateConfig(newOps);
            }}
            onDelete={() => {
              const newOps = operations.filter(op => op.id !== operation.id);
              setOperations(newOps);
              updateConfig(newOps);
            }}
            onMoveUp={index > 0 ? () => moveOperation(index, index - 1) : undefined}
            onMoveDown={index < operations.length - 1 ? () => moveOperation(index, index + 1) : undefined}
          />
        ))}

        {/* Add Operation Buttons */}
        <div className="add-operation">
          <Button 
            variant="secondary" 
            onClick={() => addOperation('pick')}
            icon="filter"
          >
            Pick Fields
          </Button>
          <Button 
            variant="secondary" 
            onClick={() => addOperation('rename')}
            icon="edit"
          >
            Rename Field
          </Button>
          <Button 
            variant="secondary" 
            onClick={() => addOperation('compute')}
            icon="calculator"
          >
            Compute Field
          </Button>
          <Button 
            variant="secondary" 
            onClick={() => addOperation('filter')}
            icon="filter"
          >
            Filter Data
          </Button>
        </div>
      </div>

      {/* Output Schema Display */}
      <div className="schema-panel">
        <h4>Output Data</h4>
        <SchemaViewer schema={computeOutputSchema(inputSchema, operations)} />
      </div>

    </div>
  );
}
```

### Operation Card Components

```tsx
interface OperationCard {
  operation: TransformOperation;
  index: number;
  inputSchema: JSONSchema;
  onUpdate: (operation: TransformOperation) => void;
  onDelete: () => void;
  onMoveUp?: () => void;
  onMoveDown?: () => void;
}

function OperationCard({ operation, index, inputSchema, onUpdate, onDelete, onMoveUp, onMoveDown }: OperationCard) {
  return (
    <Card className="operation-card">
      
      {/* Header */}
      <CardHeader>
        <div className="operation-header">
          <Icon name={getOperationIcon(operation.type)} />
          <span className="operation-title">{getOperationTitle(operation.type)}</span>
          <div className="operation-actions">
            {onMoveUp && <Button size="sm" variant="ghost" onClick={onMoveUp}><Icon name="arrow-up" /></Button>}
            {onMoveDown && <Button size="sm" variant="ghost" onClick={onMoveDown}><Icon name="arrow-down" /></Button>}
            <Button size="sm" variant="ghost" onClick={onDelete}><Icon name="trash" /></Button>
          </div>
        </div>
      </CardHeader>

      {/* Configuration */}
      <CardBody>
        {operation.type === 'pick' && (
          <PickFieldsConfig 
            fields={operation.config.fields}
            availableFields={Object.keys(inputSchema.properties)}
            onChange={(fields) => onUpdate({ 
              ...operation, 
              config: { ...operation.config, fields } 
            })}
          />
        )}

        {operation.type === 'rename' && (
          <RenameFieldConfig 
            fromField={operation.config.from}
            toField={operation.config.to}
            availableFields={Object.keys(inputSchema.properties)}
            onChange={(from, to) => onUpdate({ 
              ...operation, 
              config: { ...operation.config, from, to } 
            })}
          />
        )}

        {operation.type === 'compute' && (
          <ComputeFieldConfig 
            fieldName={operation.config.field}
            formula={operation.config.formula}
            availableFields={Object.keys(inputSchema.properties)}
            onChange={(field, formula) => onUpdate({ 
              ...operation, 
              config: { ...operation.config, field, formula } 
            })}
          />
        )}

        {operation.type === 'filter' && (
          <FilterConfig 
            conditions={operation.config.conditions}
            availableFields={Object.keys(inputSchema.properties)}
            onChange={(conditions) => onUpdate({ 
              ...operation, 
              config: { ...operation.config, conditions } 
            })}
          />
        )}
      </CardBody>

    </Card>
  );
}
```

## 2. Formula Builder Component

```tsx
interface FormulaBuilder {
  formula?: Formula;
  availableFields: string[];
  onChange: (formula: Formula) => void;
}

function FormulaBuilder({ formula, availableFields, onChange }: FormulaBuilder) {
  const [expression, setExpression] = useState<FormulaExpression[]>(formula?.expression || []);

  const addToken = (token: FormulaToken) => {
    const newExpression = [...expression, token];
    setExpression(newExpression);
    onChange({ expression: newExpression });
  };

  return (
    <div className="formula-builder">
      
      {/* Expression Display */}
      <div className="formula-expression">
        {expression.map((token, index) => (
          <FormulaToken 
            key={index}
            token={token}
            onEdit={(newToken) => {
              const newExpr = [...expression];
              newExpr[index] = newToken;
              setExpression(newExpr);
              onChange({ expression: newExpr });
            }}
            onDelete={() => {
              const newExpr = expression.filter((_, i) => i !== index);
              setExpression(newExpr);
              onChange({ expression: newExpr });
            }}
          />
        ))}
        
        {expression.length === 0 && (
          <div className="formula-placeholder">
            Click fields and operators to build your formula
          </div>
        )}
      </div>

      {/* Field Selector */}
      <div className="formula-inputs">
        <div className="field-list">
          <h5>Available Fields</h5>
          {availableFields.map(field => (
            <Button 
              key={field}
              variant="outline" 
              size="sm"
              onClick={() => addToken({ type: 'field', value: field })}
            >
              {field}
            </Button>
          ))}
        </div>

        {/* Operators */}
        <div className="operators">
          <h5>Operators</h5>
          <div className="operator-grid">
            {['+', '-', '*', '/', '(', ')'].map(op => (
              <Button 
                key={op}
                variant="outline" 
                size="sm"
                onClick={() => addToken({ type: 'operator', value: op })}
              >
                {op}
              </Button>
            ))}
          </div>
        </div>

        {/* Functions */}
        <div className="functions">
          <h5>Functions</h5>
          {['Math.round', 'Math.max', 'Math.min', 'String.toUpperCase'].map(func => (
            <Button 
              key={func}
              variant="outline" 
              size="sm"
              onClick={() => addToken({ type: 'function', value: func })}
            >
              {func}
            </Button>
          ))}
        </div>
      </div>

      {/* Preview */}
      <div className="formula-preview">
        <h5>Generated Code</h5>
        <CodePreview 
          code={generateFormulaCode(expression)}
          language="javascript"
          readonly
        />
      </div>

    </div>
  );
}
```

## 3. Hybrid Editor Layout

```tsx
interface HybridEditor {
  workflow: WorkflowDefinition;
  onWorkflowChange: (workflow: WorkflowDefinition) => void;
}

function HybridEditor({ workflow, onWorkflowChange }: HybridEditor) {
  const [selectedElement, setSelectedElement] = useState<WorkflowElement | null>(null);
  const [showCodePanel, setShowCodePanel] = useState(false);

  return (
    <div className="hybrid-editor">
      
      {/* Main Layout */}
      <SplitPane 
        split="horizontal" 
        defaultSize="70%" 
        allowResize={true}
      >
        
        {/* Visual Editor Pane */}
        <div className="visual-pane">
          <WorkflowCanvas 
            workflow={workflow}
            selectedElement={selectedElement}
            onElementSelect={setSelectedElement}
            onElementChange={(element) => {
              // Update workflow and sync to code
              const updatedWorkflow = updateWorkflowElement(workflow, element);
              onWorkflowChange(updatedWorkflow);
            }}
            onComplexEdit={(element) => {
              // Open inline code editor for complex operations
              setSelectedElement(element);
              setShowCodePanel(true);
            }}
          />
        </div>

        {/* Code/Properties Pane */}
        <SplitPane split="vertical" defaultSize="60%">
          
          {/* Properties Panel */}
          <div className="properties-pane">
            {selectedElement && (
              <SmartPropertyPanel 
                element={selectedElement}
                onChange={(updates) => {
                  const updatedElement = { ...selectedElement, ...updates };
                  setSelectedElement(updatedElement);
                  
                  const updatedWorkflow = updateWorkflowElement(workflow, updatedElement);
                  onWorkflowChange(updatedWorkflow);
                }}
                onOpenCodeEditor={() => setShowCodePanel(true)}
              />
            )}
          </div>

          {/* Code Panel */}
          <div className="code-pane">
            {showCodePanel && selectedElement && (
              <InlineCodeEditor 
                element={selectedElement}
                context={getElementContext(workflow, selectedElement)}
                onChange={(code) => {
                  const updatedElement = updateElementCode(selectedElement, code);
                  setSelectedElement(updatedElement);
                  
                  const updatedWorkflow = updateWorkflowElement(workflow, updatedElement);
                  onWorkflowChange(updatedWorkflow);
                }}
                onClose={() => setShowCodePanel(false)}
              />
            )}
          </div>

        </SplitPane>

      </SplitPane>

      {/* AI Assistant */}
      <AIAssistantPanel 
        context={{ workflow, selectedElement }}
        onSuggestion={(suggestion) => {
          if (suggestion.type === 'code') {
            // Apply code suggestion
            const updatedWorkflow = applySuggestion(workflow, suggestion);
            onWorkflowChange(updatedWorkflow);
          }
        }}
      />

    </div>
  );
}
```

## 4. Mode Transition Indicators

```tsx
interface ElementComplexityIndicator {
  element: WorkflowElement;
  complexity: 'simple' | 'medium' | 'complex';
}

function ElementComplexityIndicator({ element, complexity }: ElementComplexityIndicator) {
  const getIndicatorConfig = () => {
    switch (complexity) {
      case 'simple':
        return {
          color: 'green',
          icon: 'check-circle',
          tooltip: 'Fully editable in visual mode',
          badge: null
        };
      case 'medium':
        return {
          color: 'orange',
          icon: 'code',
          tooltip: 'Contains custom logic - hybrid editing available',
          badge: 'Code'
        };
      case 'complex':
        return {
          color: 'red',
          icon: 'warning',
          tooltip: 'Complex logic - code editing required',
          badge: 'Code Only'
        };
    }
  };

  const config = getIndicatorConfig();

  return (
    <div className={`complexity-indicator complexity-${complexity}`}>
      <Icon 
        name={config.icon} 
        color={config.color}
        size="sm"
      />
      {config.badge && (
        <Badge variant={complexity} size="xs">
          {config.badge}
        </Badge>
      )}
      <Tooltip content={config.tooltip} />
    </div>
  );
}
```

These UI components demonstrate how the Code <-> UI relationship would work in practice, providing smooth transitions between visual and code editing while maintaining the code as the source of truth.
