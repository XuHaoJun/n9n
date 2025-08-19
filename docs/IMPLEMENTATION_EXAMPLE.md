# Implementation Example: Complete Code <-> UI Flow

## Scenario: User Creates an Email Processing Workflow

This example demonstrates the complete flow from visual editing to code generation and back.

## 1. Starting in Visual Mode

User starts by dragging nodes from the palette:

```typescript
// Initial state: Empty workflow
class EmailProcessingWorkflow extends WorkflowBase {
  public build() {
    // Initially empty - will be populated by visual editor
  }
}
```

### User Actions in Visual Editor:

1. **Drags "Webhook" node** to canvas
2. **Drags "Email Validator" node** to canvas  
3. **Drags "Email Sender" node** to canvas
4. **Connects** Webhook → Validator → Sender

### Generated AST:

```typescript
interface WorkflowAST {
  nodes: [
    {
      id: 'webhook-1',
      type: 'reactiveWebhook',
      position: { x: 50, y: 100 },
      config: { path: '/api/contact', method: 'POST' }
    },
    {
      id: 'validator-1', 
      type: 'emailValidator',
      position: { x: 250, y: 100 },
      config: { strict: true }
    },
    {
      id: 'sender-1',
      type: 'emailSender', 
      position: { x: 450, y: 100 },
      config: { provider: 'sendgrid' }
    }
  ],
  connections: [
    {
      id: 'conn-1',
      from: { nodeId: 'webhook-1', slot: 'data' },
      to: { nodeId: 'validator-1', slot: 'input' },
      config: { type: 'basic' }
    },
    {
      id: 'conn-2', 
      from: { nodeId: 'validator-1', slot: 'valid' },
      to: { nodeId: 'sender-1', slot: 'messages' },
      config: { type: 'basic' }
    }
  ]
}
```

### Auto-Generated Code:

```typescript
class EmailProcessingWorkflow extends WorkflowBase {
  public build() {
    // Auto-generated from visual editor
    const webhook = this.addNode(reactiveWebhook, {
      config: { path: '/api/contact', method: 'POST' }
    });
    
    const validator = this.addNode(emailValidator, {
      config: { strict: true }
    });
    
    const sender = this.addNode(emailSender, {
      config: { provider: 'sendgrid' }
    });
    
    // Basic connections
    this.connectReactive(webhook.slots.data, validator.slots.input);
    this.connectReactive(validator.slots.valid, sender.slots.messages);
  }
}
```

## 2. Adding Simple Transform in Visual Mode

User clicks on connection between webhook and validator, wants to add data enrichment:

### Visual Transform Builder:

1. **Select "Compute Field" operation**
2. **Configure**: 
   - Field name: `receivedAt`
   - Formula: `Date.now()`
3. **Select "Rename Field" operation**
4. **Configure**:
   - From: `body.email`  
   - To: `emailAddress`

### Visual Config Generated:

```typescript
const visualTransform: VisualTransformConfig = {
  operations: [
    {
      type: 'compute',
      config: {
        field: 'receivedAt',
        formula: { expression: [{ type: 'function', value: 'Date.now()' }] }
      }
    },
    {
      type: 'rename', 
      config: {
        from: 'body.email',
        to: 'emailAddress'
      }
    }
  ]
};
```

### Generated Transform Code:

```typescript
// Auto-generated from visual builder
this.connectReactive(webhook.slots.data, validator.slots.input, {
  transform: (data) => {
    // Compute receivedAt
    const step1 = { ...data, receivedAt: Date.now() };
    
    // Rename body.email to emailAddress
    const step2 = { ...step1, emailAddress: step1.body?.email };
    delete step2.body?.email;
    
    return step2;
  }
});
```

## 3. User Needs Complex Logic - Transition to Code

User realizes they need more complex validation logic that visual editor can't handle.

### UI Indicators:

- Connection shows **orange badge**: "Custom Logic Needed"
- Property panel shows **"Switch to Code Editor"** button
- Tooltip: "This logic is too complex for visual editing"

### User Clicks "Edit Code":

Inline code editor opens with current generated code:

```typescript
// Editable in inline code editor
transform: (data) => {
  // Simple operations (generated from visual)
  const enriched = { ...data, receivedAt: Date.now() };
  const renamed = { ...enriched, emailAddress: enriched.body?.email };
  
  // User adds complex logic here
  if (!renamed.emailAddress || !isValidEmailDomain(renamed.emailAddress)) {
    throw new ValidationError('Invalid email domain');
  }
  
  // Complex business logic
  const userProfile = await enrichUserProfile(renamed.emailAddress);
  const riskScore = calculateRiskScore(userProfile, renamed.body);
  
  if (riskScore > 0.8) {
    return {
      ...renamed,
      requiresManualReview: true,
      riskScore,
      reviewReason: 'High risk score detected'
    };
  }
  
  return {
    ...renamed,
    riskScore,
    processedAt: Date.now(),
    userProfile: {
      segment: userProfile.segment,
      isExistingCustomer: userProfile.exists
    }
  };
}
```

### AST Update:

```typescript
// Connection now marked as having complex logic
{
  id: 'conn-1',
  from: { nodeId: 'webhook-1', slot: 'data' },
  to: { nodeId: 'validator-1', slot: 'input' },
  config: { 
    type: 'complex',
    hasCustomTransform: true,
    transformCode: "...", // Full function code
    visualPreview: {
      operations: [/* original visual operations */],
      hasAdditionalLogic: true
    }
  }
}
```

### Visual Representation Updates:

- Connection line changes to **orange/red** indicating complexity
- **Code icon** appears on connection
- **Code preview tooltip** shows first few lines
- **Complexity indicator**: "Contains custom logic"

## 4. Code-First Development

Developer decides to add a new node directly in code:

### User Types in Code Editor:

```typescript
class EmailProcessingWorkflow extends WorkflowBase {
  public build() {
    // Existing nodes...
    const webhook = this.addNode(reactiveWebhook, {
      config: { path: '/api/contact', method: 'POST' }
    });
    
    const validator = this.addNode(emailValidator, {
      config: { strict: true }
    });
    
    // NEW: User adds analytics node via code
    const analytics = this.addNode(analyticsCollector, {
      config: { 
        metrics: ['email_received', 'validation_result'],
        aggregationWindow: 60000 
      }
    });
    
    const sender = this.addNode(emailSender, {
      config: { provider: 'sendgrid' }
    });
    
    // Existing connections...
    this.connectReactive(webhook.slots.data, validator.slots.input, {
      transform: /* complex transform from above */
    });
    
    // NEW: Fan-out pattern for analytics
    this.fanOut(webhook.slots.data, [
      {
        target: analytics.slots.events,
        transform: (data) => ({
          event: 'email_received',
          timestamp: Date.now(),
          metadata: { source: 'webhook', path: data.path }
        })
      }
    ]);
    
    this.connectReactive(validator.slots.valid, sender.slots.messages);
    this.connectReactive(validator.slots.invalid, analytics.slots.events, {
      transform: (data) => ({
        event: 'validation_failed',
        timestamp: Date.now(),
        reason: data.validationError
      })
    });
  }
}
```

### Code Parser Detects Changes:

```typescript
const codeChanges: CodeChange[] = [
  {
    type: 'node_added',
    node: {
      id: 'analytics-1',
      type: 'analyticsCollector',
      position: null, // Will be auto-positioned
      config: { /* parsed from code */ }
    }
  },
  {
    type: 'connection_added',
    connection: {
      id: 'conn-3',
      from: { nodeId: 'webhook-1', slot: 'data' },
      to: { nodeId: 'analytics-1', slot: 'events' },
      config: { type: 'complex', hasCustomTransform: true }
    }
  },
  {
    type: 'fanout_pattern_detected',
    sourceNode: 'webhook-1',
    targets: ['validator-1', 'analytics-1']
  }
];
```

### Visual Editor Updates:

1. **New node appears** on canvas (auto-positioned)
2. **New connections** are drawn with appropriate styling
3. **Fan-out pattern** is visually represented with branching
4. **Complexity indicators** show which connections have custom code

## 5. Collaborative Editing

Two developers work on the same workflow:

### Developer A (Visual Mode):
- Adjusts node positions
- Changes email provider configuration
- Adds rate limiting via UI controls

### Developer B (Code Mode):  
- Adds error handling logic
- Implements retry mechanism
- Adds custom metrics

### Conflict Resolution:

```typescript
class ConflictResolver {
  resolve(visualChanges: UIChange[], codeChanges: CodeChange[]): Resolution {
    return {
      // Visual changes win for layout and simple config
      applyFromVisual: [
        'node_position_changes',
        'basic_configuration_changes'
      ],
      
      // Code changes win for complex logic
      applyFromCode: [
        'custom_transform_changes',
        'error_handling_changes',
        'new_method_definitions'
      ],
      
      // Merge strategy for compatible changes
      merge: [
        'connection_configuration' // Merge visual config with code logic
      ],
      
      // Requires manual resolution
      conflicts: [
        // When both modify the same transform function
      ]
    };
  }
}
```

## 6. Final Generated Workflow

After all edits, the complete workflow:

### TypeScript Code:

```typescript
class EmailProcessingWorkflow extends WorkflowBase {
  public build() {
    const webhook = this.addNode(reactiveWebhook, {
      config: { path: '/api/contact', method: 'POST' }
    });
    
    const validator = this.addNode(emailValidator, {
      config: { strict: true }
    });
    
    const analytics = this.addNode(analyticsCollector, {
      config: { 
        metrics: ['email_received', 'validation_result'],
        aggregationWindow: 60000 
      }
    });
    
    const sender = this.addNode(emailSender, {
      config: { 
        provider: 'sendgrid',
        rateLimit: 10 // Added via visual editor
      }
    });
    
    // Complex transform with visual + custom code
    this.connectReactive(webhook.slots.data, validator.slots.input, {
      transform: async (data) => {
        // Visual operations (maintained from original)
        const enriched = { ...data, receivedAt: Date.now() };
        const renamed = { ...enriched, emailAddress: enriched.body?.email };
        
        // Custom validation logic (added via code)
        if (!renamed.emailAddress || !isValidEmailDomain(renamed.emailAddress)) {
          throw new ValidationError('Invalid email domain');
        }
        
        const userProfile = await enrichUserProfile(renamed.emailAddress);
        const riskScore = calculateRiskScore(userProfile, renamed.body);
        
        return {
          ...renamed,
          riskScore,
          requiresManualReview: riskScore > 0.8,
          userProfile: { segment: userProfile.segment }
        };
      },
      
      // Error handling (added via code)
      onError: async (error, data) => {
        await this.logError('Transform failed', { error, data });
        return { ...data, validationError: error.message };
      },
      
      // Rate limiting (added via visual editor)  
      rateLimit: { requests: 100, windowMs: 1000 }
    });
    
    // Fan-out analytics tracking
    this.fanOut(webhook.slots.data, [
      {
        target: analytics.slots.events,
        transform: (data) => ({
          event: 'email_received',
          timestamp: Date.now(),
          metadata: { source: 'webhook' }
        })
      }
    ]);
    
    this.connectReactive(validator.slots.valid, sender.slots.messages);
    this.connectReactive(validator.slots.invalid, analytics.slots.events, {
      transform: (data) => ({
        event: 'validation_failed',
        timestamp: Date.now(),
        reason: data.validationError
      })
    });
  }
}
```

### Visual Representation:

- **4 nodes** with proper positioning
- **4 connections** with different complexity indicators
- **Fan-out pattern** visually represented
- **Mixed editing indicators**: Some green (visual), some orange (hybrid), some red (code-only)

This example demonstrates how the Code <-> UI relationship enables seamless transitions between editing modes while maintaining code as the source of truth and providing appropriate visual feedback for different complexity levels.
