# n9n

## 簡介

n9n 是一款專為開發者從零開始打造的次世代工作流自動化平台。我們摒棄了傳統基於 JSON 的笨重配置，將工作流視為可測試、可版本化且高度可維護的一流軟體資產。

### 主要特色

- 工作流即程式碼 (Workflow as Code)：您的自動化流程就是 TypeScript/JavaScript 程式碼。享受完整的 Git 版本控制、單元測試、程式碼重用以及您熟悉的所有開發工具鏈帶來的強大能力。

- 三位一體的開發體驗 (Unified Development Experience)：從 No-Code 視覺化拖曳、AI 輔助生成 到 Pro-Code 的精準控制，n9n 在三種模式間無縫轉換。無論是快速原型設計還是打造企業級的複雜流程，您都能找到最高效的路徑。

- 響應式函數節點 (Reactive Function Nodes)：每個節點都是一個獨立、響應式的函數。這種清晰的抽象化不僅讓除錯變得直觀，更讓您能以函數式編程的思維，優雅地處理複雜的非同步數據流。

- 高效能 ECS 架構 (High-Performance ECS Architecture)：底層採用實體組件系統 (Entity Component System)，將工作流的狀態與邏輯解耦。這不僅帶來了卓越的執行效能與水平擴展能力，也為未來的視覺化編輯與即時監控奠定了堅實基礎。

n9n 不僅僅是一個工具，它是一種全新的理念：將嚴謹的軟體工程實踐注入到自動化領域。我們賦予開發者前所未有的自由度與控制力，讓您能建構出真正穩定、可擴展且經得起時間考驗的自動化系統。

## Introduction

n9n is a next-generation workflow automation platform built from the ground up for developers. We've abandoned the cumbersome JSON-based configurations of traditional tools, instead treating workflows as first-class software assets that are testable, versionable, and highly maintainable.

### Key Features

- **Workflow as Code**: Your automation processes are TypeScript/JavaScript code. Enjoy the full power of Git version control, unit testing, code reuse, and all the development toolchain you're familiar with.

- **Unified Development Experience**: Seamlessly transition between No-Code visual drag-and-drop, AI-assisted generation, and Pro-Code precise control. Whether you're rapid prototyping or building enterprise-grade complex processes, you'll find the most efficient path.

- **Reactive Function Nodes**: Each node is an independent, reactive function. This clean abstraction not only makes debugging intuitive but also enables you to handle complex asynchronous data flows elegantly with functional programming principles.

- **High-Performance ECS Architecture**: Built on Entity Component System foundation, decoupling workflow state from logic. This delivers exceptional execution performance and horizontal scaling capabilities, while establishing a solid foundation for future visual editing and real-time monitoring.

n9n is more than just a tool—it's a new paradigm: injecting rigorous software engineering practices into the automation domain. We empower developers with unprecedented freedom and control, enabling you to build truly stable, scalable, and time-tested automation systems.

## 📚 Design Documentation

For detailed technical design and architecture:

- **[Code <-> UI Relationship Design](./docs/CODE_UI_RELATIONSHIP.md)** - Complete architecture for bidirectional code/visual synchronization
- **[UI Component Examples](./docs/UI_COMPONENT_EXAMPLES.md)** - Practical UI component implementations for different editing modes
- **[Implementation Example](./docs/IMPLEMENTATION_EXAMPLE.md)** - End-to-end example showing visual to code workflow

## Reactive Slot Connections - Advanced Workflow Design

### 1. Basic Reactive Node Definitions

```javascript
// Reactive webhook trigger
@NodeInfo({ uuid: "webhook-reactive", displayName: "Reactive Webhook" })
@Position({ x: 50, y: 100 })
@OutputSlots({
  data: {
    type: 'stream',
    reactive: true,
    displayName: 'Request Stream',
    color: '#2196F3',
    icon: 'stream',
    position: 'right'
  }
})
function reactiveWebhook(@OutputSlots() outputs, @SelfEntity() self) {
  // Create reactive stream from webhook events
  const requestStream = createWebhookStream(self.config.path);

  // Emit each incoming request
  requestStream.subscribe(request => {
    outputs.data.emit({
      body: request.body,
      headers: request.headers,
      timestamp: Date.now(),
      id: generateId()
    });
  });

  return {
    dispose: () => requestStream.unsubscribe()
  };
}

// Reactive data processor
@NodeInfo({ uuid: "processor-reactive", displayName: "Stream Processor" })
@Position({ x: 250, y: 100 })
@InputSlots({
  stream: {
    type: 'stream',
    reactive: true,
    displayName: 'Input Stream',
    color: '#2196F3',
    icon: 'stream',
    position: 'left',
    required: true
  },
  config: {
    type: 'object',
    reactive: false,
    displayName: 'Configuration',
    color: '#FF9800',
    icon: 'settings',
    position: 'top',
    default: { batchSize: 10, timeout: 5000 }
  }
})
@OutputSlots({
  processed: {
    type: 'stream',
    reactive: true,
    displayName: 'Processed Stream',
    color: '#4CAF50',
    icon: 'check-circle',
    position: 'right'
  },
  errors: {
    type: 'stream',
    reactive: true,
    displayName: 'Error Stream',
    color: '#F44336',
    icon: 'error',
    position: 'bottom'
  },
  stats: {
    type: 'object',
    reactive: true,
    throttle: 1000, // Emit max once per second
    displayName: 'Statistics',
    color: '#9C27B0',
    icon: 'analytics',
    position: 'bottom'
  }
})
function streamProcessor(@InputSlots() inputs, @OutputSlots() outputs) {
  let processedCount = 0;
  let errorCount = 0;

  // React to streaming data with backpressure
  inputs.stream
    .buffer(inputs.config.batchSize)
    .timeout(inputs.config.timeout)
    .subscribe({
      next: (batch) => {
        try {
          const processed = batch.map(item => ({
            ...item,
            processed: true,
            processedAt: Date.now(),
            batchId: generateId()
          }));

          // Emit processed batch
          outputs.processed.emit(processed);
          processedCount += processed.length;

          // Emit stats (throttled)
          outputs.stats.emit({
            processed: processedCount,
            errors: errorCount,
            rate: processedCount / (Date.now() - startTime) * 1000
          });

        } catch (error) {
          outputs.errors.emit({
            error: error.message,
            batch: batch,
            timestamp: Date.now()
          });
          errorCount++;
        }
      },
      error: (error) => {
        outputs.errors.emit({
          error: 'Stream processing failed',
          details: error.message,
          timestamp: Date.now()
        });
      }
    });
}

// Reactive email sender
@NodeInfo({ uuid: "email-reactive", displayName: "Reactive Email Sender" })
@Position({ x: 450, y: 100 })
@InputSlots({
  messages: {
    type: 'stream',
    reactive: true,
    displayName: 'Message Stream',
    color: '#4CAF50',
    icon: 'email',
    position: 'left',
    required: true
  },
  config: {
    type: 'object',
    reactive: false,
    displayName: 'Email Config',
    color: '#FF9800',
    icon: 'settings',
    position: 'top',
    default: { rateLimit: 10, retries: 3 }
  }
})
@OutputSlots({
  sent: {
    type: 'stream',
    reactive: true,
    displayName: 'Sent Confirmations',
    color: '#4CAF50',
    icon: 'check',
    position: 'right'
  },
  failed: {
    type: 'stream',
    reactive: true,
    displayName: 'Failed Sends',
    color: '#F44336',
    icon: 'error',
    position: 'bottom'
  }
})
function reactiveEmailSender(@InputSlots() inputs, @OutputSlots() outputs) {
  // Rate-limited email sending with retries
  inputs.messages
    .rateLimit(inputs.config.rateLimit) // Limit sends per second
    .retry(inputs.config.retries)
    .subscribe({
      next: async (message) => {
        try {
          const result = await sendEmail({
            to: message.email,
            subject: message.subject,
            body: message.body,
            template: message.template
          });

          outputs.sent.emit({
            messageId: result.messageId,
            originalData: message,
            sentAt: Date.now(),
            provider: result.provider
          });

        } catch (error) {
          outputs.failed.emit({
            error: error.message,
            originalData: message,
            failedAt: Date.now(),
            retryCount: error.retryCount || 0
          });
        }
      },
      error: (error) => {
        outputs.failed.emit({
          error: 'Email service failed',
          details: error.message,
          timestamp: Date.now()
        });
      }
    });
}
```

### 2. Complete Reactive Workflow Building

```javascript
class ReactiveEmailWorkflow extends WorkflowBase {
  // will directly run in frontend and backend,
  // frontend will get graph(nodes/edges) for render no code editor!
  // backend will exec it!
  public build() {
    // 1. Create reactive nodes
    const webhook = this.addNode(reactiveWebhook, {
      config: { path: '/api/contact', method: 'POST' }
    });

    const validator = this.addNode(dataValidator, {
      config: {
        schema: ContactFormSchema,
        strict: true
      }
    });

    const processor = this.addNode(streamProcessor, {
      config: {
        batchSize: 5,
        timeout: 3000
      }
    });

    const emailSender = this.addNode(reactiveEmailSender, {
      config: {
        rateLimit: 2, // 2 emails per second
        retries: 3
      }
    });

    const logger = this.addNode(streamLogger, {
      config: {
        level: 'info',
        format: 'json'
      }
    });

    const metrics = this.addNode(metricsCollector, {
      config: {
        interval: 5000,
        retention: '1h'
      }
    });

    // 2. Connect reactive streams with advanced configurations
    this.connectReactive(webhook.slots.data, validator.slots.input, {
      // Transform incoming data
      transform: (data) => ({
        ...data,
        received_at: Date.now(),
        source: 'webhook'
      }),

      // Filter invalid requests
      filter: (data) => data.body && data.body.email,

      // Backpressure handling
      backpressure: {
        strategy: 'buffer',
        bufferSize: 1000,
        overflow: 'drop-oldest'
      },

      // Error handling
      onError: (error, data) => {
        logger.slots.errors.emit({
          stage: 'webhook-validation',
          error: error.message,
          data: data
        });
      }
    });

    // Connect validator to processor
    this.connectReactive(validator.slots.valid, processor.slots.stream, {
      // Enrich data before processing
      transform: (data) => ({
        ...data,
        priority: calculatePriority(data),
        template: selectTemplate(data)
      }),

      // Circuit breaker pattern
      circuitBreaker: {
        errorThreshold: 10,
        timeoutMs: 30000,
        resetTimeoutMs: 60000
      }
    });

    // Connect processor to email sender
    this.connectReactive(processor.slots.processed, emailSender.slots.messages, {
      // Flatten batches to individual messages
      transform: (batch) => batch.flatMap(item => ({
        email: item.body.email,
        subject: `Welcome ${item.body.name}`,
        body: item.body.message,
        template: item.template,
        priority: item.priority
      })),

      // Rate limiting at connection level
      rateLimit: {
        requests: 10,
        windowMs: 1000
      }
    });

    // Connect error streams to logger
    this.connectReactive(validator.slots.invalid, logger.slots.errors);
    this.connectReactive(processor.slots.errors, logger.slots.errors);
    this.connectReactive(emailSender.slots.failed, logger.slots.errors);

    // Connect success streams to metrics
    this.connectReactive(emailSender.slots.sent, metrics.slots.events, {
      transform: (data) => ({
        event: 'email_sent',
        timestamp: data.sentAt,
        metadata: {
          provider: data.provider,
          messageId: data.messageId
        }
      })
    });

    // Connect stats streams
    this.connectReactive(processor.slots.stats, metrics.slots.stats);

    // Multi-stream aggregation for dashboards
    this.aggregateStreams([
      processor.slots.stats,
      emailSender.slots.sent,
      emailSender.slots.failed
    ], metrics.slots.dashboard, {
      windowSize: 60000, // 1 minute windows
      aggregationFn: (stats, sent, failed) => ({
        processed_per_minute: stats.processed,
        sent_per_minute: sent.length,
        failed_per_minute: failed.length,
        success_rate: sent.length / (sent.length + failed.length),
        timestamp: Date.now()
      })
    });
  }

  public async start() {
    // Initialize all reactive streams
    await this.initializeNodes();

    // Start monitoring
    this.startHealthCheck();

    console.log('Reactive workflow started');
  }

  public async stop() {
    // Gracefully close all streams
    await this.disposeNodes();

    console.log('Reactive workflow stopped');
  }
}
```

### 3. Advanced Reactive Patterns

```javascript
// Fan-out pattern: One stream to multiple processors
class FanOutWorkflow {
  public build() {
    const source = this.addNode(dataSource);
    const processorA = this.addNode(analyticsProcessor);
    const processorB = this.addNode(emailProcessor);
    const processorC = this.addNode(webhookProcessor);

    // Fan-out with different filters
    this.fanOut(source.slots.data, [
      {
        target: processorA.slots.input,
        filter: (data) => data.type === 'analytics',
        transform: (data) => ({ ...data, processor: 'analytics' })
      },
      {
        target: processorB.slots.input,
        filter: (data) => data.type === 'email',
        transform: (data) => ({ ...data, processor: 'email' })
      },
      {
        target: processorC.slots.input,
        filter: (data) => data.type === 'webhook',
        transform: (data) => ({ ...data, processor: 'webhook' })
      }
    ]);
  }
}

// Fan-in pattern: Multiple streams to one processor
class FanInWorkflow {
  public build() {
    const sourceA = this.addNode(webhookSource);
    const sourceB = this.addNode(queueSource);
    const sourceC = this.addNode(fileSource);
    const processor = this.addNode(unifiedProcessor);

    // Fan-in with stream merging
    this.fanIn([
      {
        source: sourceA.slots.data,
        tag: 'webhook',
        priority: 1
      },
      {
        source: sourceB.slots.data,
        tag: 'queue',
        priority: 2
      },
      {
        source: sourceC.slots.data,
        tag: 'file',
        priority: 3
      }
    ], processor.slots.input, {
      mergeStrategy: 'priority', // or 'round-robin', 'timestamp'
      bufferSize: 100
    });
  }
}

// Complex stream transformation pipeline
class StreamPipelineWorkflow {
  public build() {
    const source = this.addNode(eventSource);
    const enricher = this.addNode(dataEnricher);
    const aggregator = this.addNode(windowAggregator);
    const anomalyDetector = this.addNode(anomalyDetector);
    const alerter = this.addNode(alertSystem);

    // Pipeline with windowing and complex operations
    this.pipeline([
      {
        from: source.slots.events,
        to: enricher.slots.input,
        transform: (event) => ({
          ...event,
          enriched_at: Date.now(),
          session_id: extractSessionId(event)
        })
      },
      {
        from: enricher.slots.output,
        to: aggregator.slots.input,
        window: {
          type: 'sliding',
          size: 60000, // 1 minute
          slide: 10000  // 10 seconds
        },
        groupBy: 'session_id'
      },
      {
        from: aggregator.slots.output,
        to: anomalyDetector.slots.input,
        transform: (window) => ({
          session_id: window.key,
          metrics: {
            count: window.events.length,
            avg_duration: calculateAverage(window.events, 'duration'),
            error_rate: calculateErrorRate(window.events)
          },
          timestamp: window.timestamp
        })
      },
      {
        from: anomalyDetector.slots.anomalies,
        to: alerter.slots.input,
        filter: (anomaly) => anomaly.severity > 0.7,
        transform: (anomaly) => ({
          alert_type: 'performance_anomaly',
          severity: anomaly.severity,
          description: `Anomaly detected in session ${anomaly.session_id}`,
          metrics: anomaly.metrics,
          timestamp: Date.now()
        })
      }
    ]);
  }
}
```

### 4. Real-World E-commerce Order Processing Workflow

````javascript
class EcommerceOrderWorkflow {
  public build() {
    // Order processing pipeline
    const orderReceiver = this.addNode(orderWebhook);
    const validator = this.addNode(orderValidator);
    const inventoryChecker = this.addNode(inventoryService);
    const paymentProcessor = this.addNode(paymentService);
    const fulfillmentService = this.addNode(fulfillmentService);
    const emailNotifier = this.addNode(emailService);
    const analytics = this.addNode(analyticsService);

    // Main order flow
    this.connectReactive(orderReceiver.slots.orders, validator.slots.input, {
      transform: (order) => ({
        ...order,
        received_at: Date.now(),
        order_id: generateOrderId(),
        status: 'received'
      }),
      rateLimit: { requests: 100, windowMs: 1000 }
    });

    // Valid orders go to inventory check
    this.connectReactive(validator.slots.valid, inventoryChecker.slots.check, {
      transform: (order) => ({
        ...order,
        status: 'checking_inventory'
      })
    });

    // Available inventory goes to payment
    this.connectReactive(inventoryChecker.slots.available, paymentProcessor.slots.process, {
      transform: (order) => ({
        ...order,
        status: 'processing_payment',
        inventory_reserved: true
      })
    });

    // Successful payments go to fulfillment
    this.connectReactive(paymentProcessor.slots.success, fulfillmentService.slots.fulfill, {
      transform: (order) => ({
        ...order,
        status: 'fulfilling',
        payment_confirmed: true
      })
    });

    // All status changes trigger notifications
    this.fanOut(fulfillmentService.slots.shipped, [
      {
        target: emailNotifier.slots.send,
        transform: (order) => ({
          to: order.customer.email,
          template: 'order_shipped',
          data: {
            order_id: order.order_id,
            tracking_number: order.tracking_number,
            estimated_delivery: order.estimated_delivery
          }
        })
      },
      {
        target: analytics.slots.events,
        transform: (order) => ({
          event: 'order_shipped',
          order_id: order.order_id,
          customer_id: order.customer.id,
          value: order.total,
          timestamp: Date.now()
        })
      }
    ]);

    // Error handling flows
    this.connectReactive(inventoryChecker.slots.unavailable, emailNotifier.slots.send, {
      transform: (order) => ({
        to: order.customer.email,
        template: 'inventory_unavailable',
        data: { order_id: order.order_id }
      })
    });

    this.connectReactive(paymentProcessor.slots.failed, emailNotifier.slots.send, {
      transform: (order) => ({
        to: order.customer.email,
        template: 'payment_failed',
        data: {
          order_id: order.order_id,
          reason: order.payment_failure_reason
        }
      })
    });

    // Real-time analytics aggregation
    this.aggregateStreams([
      validator.slots.valid,
      paymentProcessor.slots.success,
      fulfillmentService.slots.shipped
    ], analytics.slots.dashboard, {
      windowSize: 300000, // 5 minutes
      aggregationFn: (orders, payments, shipments) => ({
        orders_per_5min: orders.length,
        payments_per_5min: payments.length,
        shipments_per_5min: shipments.length,
        revenue_per_5min: payments.reduce((sum, p) => sum + p.total, 0),
        conversion_rate: payments.length / orders.length,
        fulfillment_rate: shipments.length / payments.length,
        timestamp: Date.now()
      })
    });
  }
}

### 5. Advanced Stream Features & Connection Types

```javascript
// Reactive connection types and configurations
class ReactiveWorkflowConnections {

  // Basic reactive connection
  basicConnect(fromSlot, toSlot) {
    return this.connectReactive(fromSlot, toSlot, {
      type: 'basic',
      autoReconnect: true,
      errorHandling: 'continue'
    });
  }

  // Buffered connection with backpressure
  bufferedConnect(fromSlot, toSlot, options = {}) {
    return this.connectReactive(fromSlot, toSlot, {
      type: 'buffered',
      bufferSize: options.bufferSize || 1000,
      bufferStrategy: options.strategy || 'circular', // 'circular', 'drop-oldest', 'drop-newest'
      flushInterval: options.flushInterval || 100,
      backpressure: {
        strategy: 'buffer',
        maxBuffer: options.maxBuffer || 5000,
        onOverflow: options.onOverflow || 'drop-oldest'
      }
    });
  }

  // Windowed connection for time-based aggregation
  windowedConnect(fromSlot, toSlot, windowConfig) {
    return this.connectReactive(fromSlot, toSlot, {
      type: 'windowed',
      window: {
        type: windowConfig.type, // 'tumbling', 'sliding', 'session'
        size: windowConfig.size,  // in milliseconds
        slide: windowConfig.slide, // for sliding windows
        sessionTimeout: windowConfig.sessionTimeout, // for session windows
        keyBy: windowConfig.keyBy // grouping function
      },
      aggregation: {
        fn: windowConfig.aggregationFn,
        emitEmpty: windowConfig.emitEmpty || false
      }
    });
  }

  // Conditional routing connection
  conditionalConnect(fromSlot, routes) {
    return routes.map(route =>
      this.connectReactive(fromSlot, route.toSlot, {
        type: 'conditional',
        condition: route.condition,
        transform: route.transform,
        priority: route.priority || 0
      })
    );
  }

  // Load-balanced connection to multiple targets
  loadBalancedConnect(fromSlot, targetSlots, strategy = 'round-robin') {
    return this.connectReactive(fromSlot, targetSlots, {
      type: 'load-balanced',
      strategy: strategy, // 'round-robin', 'least-busy', 'random', 'weighted'
      healthCheck: {
        enabled: true,
        interval: 5000,
        timeout: 1000
      },
      retries: {
        attempts: 3,
        backoff: 'exponential',
        maxDelay: 10000
      }
    });
  }
}

// Complete real-time analytics workflow
class RealTimeAnalyticsWorkflow {
  public build() {
    // Data sources
    const webEvents = this.addNode(webEventSource);
    const mobileEvents = this.addNode(mobileEventSource);
    const apiEvents = this.addNode(apiEventSource);

    // Processing nodes
    const eventEnricher = this.addNode(eventEnricherNode);
    const sessionizer = this.addNode(sessionizerNode);
    const aggregator = this.addNode(realTimeAggregator);
    const anomalyDetector = this.addNode(anomalyDetectorNode);

    // Output nodes
    const dashboard = this.addNode(dashboardUpdater);
    const alerter = this.addNode(alertingSystem);
    const warehouse = this.addNode(dataWarehouse);

    // 1. Merge all event sources
    this.fanIn([
      { source: webEvents.slots.events, tag: 'web', priority: 1 },
      { source: mobileEvents.slots.events, tag: 'mobile', priority: 1 },
      { source: apiEvents.slots.events, tag: 'api', priority: 2 }
    ], eventEnricher.slots.input, {
      mergeStrategy: 'timestamp',
      bufferSize: 1000,
      maxLatency: 1000 // 1 second max latency
    });

    // 2. Enrich events with user/session data
    this.connectReactive(eventEnricher.slots.enriched, sessionizer.slots.input, {
      transform: (event) => ({
        ...event,
        enriched_at: Date.now(),
        user_segment: getUserSegment(event.user_id),
        geo_location: getGeoLocation(event.ip)
      }),
      filter: (event) => event.user_id && event.event_type,
      onError: (error, event) => {
        this.logError('Event enrichment failed', { error, event });
      }
    });

    // 3. Sessionize events (sliding window by user)
    this.windowedConnect(sessionizer.slots.sessions, aggregator.slots.input, {
      type: 'session',
      sessionTimeout: 30 * 60 * 1000, // 30 minutes
      keyBy: (event) => event.user_id,
      aggregationFn: (events) => ({
        user_id: events[0].user_id,
        session_start: Math.min(...events.map(e => e.timestamp)),
        session_end: Math.max(...events.map(e => e.timestamp)),
        event_count: events.length,
        page_views: events.filter(e => e.event_type === 'page_view').length,
        conversions: events.filter(e => e.event_type === 'conversion').length,
        total_duration: Math.max(...events.map(e => e.timestamp)) - Math.min(...events.map(e => e.timestamp)),
        pages_visited: [...new Set(events.map(e => e.page_url))],
        user_segment: events[0].user_segment
      })
    });

    // 4. Real-time aggregations (tumbling windows)
    this.windowedConnect(aggregator.slots.metrics, dashboard.slots.update, {
      type: 'tumbling',
      size: 60 * 1000, // 1 minute windows
      aggregationFn: (sessions) => ({
        timestamp: Date.now(),
        active_users: sessions.length,
        avg_session_duration: sessions.reduce((sum, s) => sum + s.total_duration, 0) / sessions.length,
        total_page_views: sessions.reduce((sum, s) => sum + s.page_views, 0),
        total_conversions: sessions.reduce((sum, s) => sum + s.conversions, 0),
        conversion_rate: sessions.reduce((sum, s) => sum + s.conversions, 0) / sessions.reduce((sum, s) => sum + s.page_views, 0),
        top_pages: getTopPages(sessions),
        user_segments: getSegmentBreakdown(sessions)
      })
    });

    // 5. Anomaly detection on aggregated metrics
    this.connectReactive(aggregator.slots.metrics, anomalyDetector.slots.input, {
      transform: (metrics) => ({
        ...metrics,
        features: [
          metrics.active_users,
          metrics.avg_session_duration,
          metrics.conversion_rate
        ]
      })
    });

    // 6. Alert on anomalies
    this.connectReactive(anomalyDetector.slots.anomalies, alerter.slots.input, {
      filter: (anomaly) => anomaly.severity > 0.8,
      transform: (anomaly) => ({
        alert_type: 'traffic_anomaly',
        severity: anomaly.severity,
        description: `Unusual traffic pattern detected: ${anomaly.description}`,
        metrics: anomaly.original_metrics,
        timestamp: Date.now(),
        channels: ['slack', 'email', 'pagerduty']
      })
    });

    // 7. Data warehouse export (batched)
    this.bufferedConnect(aggregator.slots.metrics, warehouse.slots.input, {
      bufferSize: 100,
      flushInterval: 5 * 60 * 1000, // 5 minutes
      strategy: 'time-based',
      transform: (metricsBatch) => ({
        batch_id: generateId(),
        timestamp: Date.now(),
        metrics_count: metricsBatch.length,
        data: metricsBatch
      })
    });

    // 8. Error handling and monitoring
    this.connectReactive(eventEnricher.slots.errors, alerter.slots.input, {
      transform: (error) => ({
        alert_type: 'processing_error',
        severity: 0.6,
        description: `Event processing error: ${error.message}`,
        error_details: error,
        timestamp: Date.now(),
        channels: ['slack']
      })
    });
  }

  public async start() {
    await this.initializeNodes();
    this.startHealthMonitoring();
    console.log('Real-time analytics workflow started');
  }

  private startHealthMonitoring() {
    setInterval(() => {
      const health = this.getSystemHealth();
      if (health.status !== 'healthy') {
        this.emit('health_alert', health);
      }
    }, 30000); // Check every 30 seconds
  }
}
````

## Frontend Reactive Slot Visualization

```javascript
// Enhanced reactive slot rendering with real-time data flow
function renderReactiveWorkflow(workflow) {
  const nodes = workflow.getNodes();
  const connections = workflow.getConnections();

  nodes.forEach((node) => {
    const element = createNodeElement(node);

    // Render input slots with reactive indicators
    node.inputSlots.forEach((slot) => {
      const slotElement = createSlotElement(slot, {
        position: slot.position,
        color: slot.color,
        icon: slot.icon,
        reactive: slot.reactive,
        // Show data flow rate if reactive
        dataRate: slot.reactive ? getSlotDataRate(slot) : null,
        // Show buffer status
        bufferStatus: slot.bufferSize ? getBufferStatus(slot) : null,
      });

      element.appendChild(slotElement);
    });

    // Render output slots
    node.outputSlots.forEach((slot) => {
      const slotElement = createSlotElement(slot, {
        position: slot.position,
        color: slot.color,
        icon: slot.icon,
        reactive: slot.reactive,
        // Show real-time metrics
        emissionRate: slot.reactive ? getEmissionRate(slot) : null,
        subscriberCount: slot.reactive ? getSubscriberCount(slot) : null,
      });

      element.appendChild(slotElement);
    });
  });

  // Render connections with data flow animation
  connections.forEach((connection) => {
    const connectionElement = createConnectionElement(connection, {
      animated: connection.reactive,
      dataFlow: connection.reactive ? getConnectionDataFlow(connection) : null,
      health: getConnectionHealth(connection),
    });
  });
}

// Real-time connection monitoring
function monitorConnections(workflow) {
  workflow.getConnections().forEach((connection) => {
    if (connection.reactive) {
      // Monitor data flow rate
      connection.onDataFlow((data, rate) => {
        updateConnectionVisualization(connection.id, { rate, data });
      });

      // Monitor errors
      connection.onError((error) => {
        showConnectionError(connection.id, error);
      });

      // Monitor backpressure
      connection.onBackpressure((pressure) => {
        updateConnectionPressure(connection.id, pressure);
      });
    }
  });
}
```

```json
{
  "nodes": [
    {
      "nodeInfoComponent": {
        "uuid": "reactive-webhook",
        "displayName": "Reactive Webhook"
      },
      "positionComponent": { "x": 50, "y": 100 },
      "slotMetadataComponent": {
        "inputs": [],
        "outputs": [
          {
            "name": "data",
            "type": "stream",
            "reactive": true,
            "color": "#2196F3",
            "icon": "stream",
            "position": "right",
            "realTimeMetrics": {
              "emissionRate": "45/sec",
              "subscriberCount": 3,
              "bufferStatus": "healthy"
            }
          }
        ]
      }
    }
  ],
  "connections": [
    {
      "from": { "nodeId": "reactive-webhook", "slot": "data" },
      "to": { "nodeId": "stream-processor", "slot": "stream" },
      "type": "reactive",
      "connectionConfig": {
        "backpressure": { "strategy": "buffer", "bufferSize": 1000 },
        "transform": "data enrichment",
        "errorHandling": "continue"
      },
      "realTimeStatus": {
        "dataFlowRate": "42/sec",
        "health": "healthy",
        "latency": "12ms",
        "bufferUtilization": "23%"
      }
    }
  ]
}
```
