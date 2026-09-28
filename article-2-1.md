# Customer Support Agent

The hardest part of automating customer support is not generating a good answer. It is remembering which good answer worked for this customer before—and knowing when that old answer should influence what I do now.

That is the problem I designed this customer support agent around: make customer history an operational input instead of a pile of records that an agent has to rediscover on every ticket.

## What I built

I built the system as a web application with a React frontend, a TypeScript/Express backend, an LLM integration, and a persistent memory layer. I deliberately kept those responsibilities separate rather than putting everything behind one large “agent” abstraction.

The frontend is organized around operational views: customer support, customer profiles, customer memory, a memory explorer, AI activity, settings, and related business workflows. A small API layer connects those views to the backend. The server owns the model integration and the application logic, while the domain types define the objects that move through the system: customers, tickets, memories, traces, vendors, invoices, and business actions.

That separation matters because the interesting state in support is not the chat transcript.

A customer has an environment. They have previous incidents. They have solutions that worked, solutions that failed, escalation history, and preferences about how technical information should be communicated. I wanted those facts to be explicit in the application model.

For example, the memory categories are not just arbitrary metadata:

```ts
export type MemoryCategory =
  | 'known_issue'
  | 'environment'
  | 'troubleshooting_step'
  | 'solution_worked'
  | 'escalation_history'
  | 'customer_sentiment'
  | 'communication_style'
  | 'vendor_terms'
  | 'common_discrepancy'
  | 'historical_exception'
  | 'approval_decision'
  | 'invoice_pattern'
  | 'previous_resolution';
```

I prefer this to a generic `metadata` object. Support memory is heterogeneous, and the category tells the rest of the system what kind of fact it is handling.

A memory record also carries confidence, importance, keywords, a reference to the originating business record, access information, and similarity information:

```ts
export interface MemoryRecord {
  id: string;
  targetType: 'customer' | 'vendor';
  targetId: string;
  targetName: string;
  category: MemoryCategory;
  title: string;
  content: string;
  confidence: number;
  importance: 'critical' | 'high' | 'medium' | 'low';
  keywords: string[];
  referenceId?: string;
  createdAt: string;
  accessCount: number;
  lastRecalledAt?: string;
  similarityScore?: number;
}
```

That gives the memory layer more structure than simply storing old conversations and hoping semantic search finds something useful later.

## The real problem was memory, not chat

When I first looked at this architecture, it would have been easy to describe it as a chatbot with customer data attached.

I think that framing misses the difficult part.

A conventional support chatbot answers the current message using the current conversation and whatever documentation it can retrieve. A useful support agent needs another dimension: longitudinal context.

Imagine a customer reports:

> “ERP synchronization is not working again.”

A stateless system can search documentation for “ERP synchronization” and produce a plausible troubleshooting checklist.

A memory-aware system can ask a better question internally: have I seen this customer experience this exact failure before?

Suppose the customer runs SAP S/4HANA in a single-tenant AWS environment. Previous tickets show that the connector and Redis token cache are involved in the failure. More importantly, an earlier incident established that refreshing the SAP Gateway authentication token and flushing the Redis token cache solved the problem. That incident also recorded that restarting the daemon could cause queued synchronization packets to be dropped.

That distinction—what worked and what failed—is the core of the system.

The application therefore behaves more like a memory pipeline than a chat pipeline:

1. Receive the customer's message.
2. Identify the customer and current operational context.
3. Recall relevant long-term memories.
4. Combine those memories with the current conversation.
5. Generate a response and, when appropriate, a business action.
6. Record genuinely useful new information.
7. Expose a trace showing what happened.

For the persistent memory layer, I use [Hindsight GitHub](https://github.com/vectorize-io/hindsight). I wanted the application to treat long-term memory as infrastructure rather than turning the application process itself into the memory database.

The [Hindsight documentation](https://hindsight.vectorize.io/) is useful for the same reason: it treats agent memory as a distinct system concern rather than simply another vector-search feature.

## Why Hindsight belongs outside the model

The important benefit of a dedicated memory system is not merely persistence.

Persistence without useful retrieval just creates a larger pile of irrelevant facts.

The support agent needs memory that survives individual conversations while remaining useful in the current situation. It has to distinguish durable customer facts from transient chat messages and connect remembered information to the event that produced it.

That is where the idea of [agent memory for production systems](https://vectorize.io/what-is-agent-memory) becomes concrete.

I want the memory layer to answer questions such as:

- Has this customer seen this failure before?
- What solution worked last time?
- What solution failed?
- What environment was involved?
- Is there a known exception?
- How does this customer prefer technical information to be communicated?
- Is the current problem similar enough to an older incident to reuse its resolution?

The domain model reflects those questions directly. `solution_worked`, `known_issue`, `environment`, `customer_sentiment`, and `communication_style` are not interchangeable facts.

I also wanted the memory system to remain independent of the LLM provider. The model configuration is treated as application configuration rather than something embedded in the frontend or business domain:

```ts
let settings: SystemSettings = {
  llmProvider: 'gemini',
  modelName: 'gemini-3.8-flash',
  similarityThreshold: 0.65,
  autoActionThreshold: 0.85,
  memoryRetentionDays: 365,
  autonomousTicketResolutions: true,
  autonomousInvoiceHolds: true,
};
```

The exact model can change without changing what the company remembers.

That is an important boundary for me. Memory is application infrastructure. The model is replaceable inference.

## The API returns more than an answer

One of the design decisions I care about most is that the support API does not return only generated text.

The client receives the response, the memories retrieved for that request, and a pipeline trace:

```ts
async sendCustomerSupportMessage(
  customerId: string,
  message: string,
  conversationHistory: any[] = [],
): Promise<{
  reply: string;
  pipelineTrace: PipelineTrace;
  retrievedMemories: MemoryRecord[];
}> {
  const res = await fetch(`${API_BASE}/support/chat`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ customerId, message, conversationHistory }),
  });

  if (!res.ok) throw new Error('Failed to send support message');
  return res.json();
}
```

That return shape is intentional.

If the UI receives only an answer, debugging becomes guesswork. I want to know which memories influenced the response. I want to know what historical context was assembled. I want to know whether the system proposed an action and whether that action was executed or held for human confirmation.

That is what `PipelineTrace` represents:

```ts
export interface PipelineTrace {
  id: string;
  timestamp: string;
  agentType: 'Customer Support' | 'Accounts Payable';
  currentInput: string;
  retrievedMemories: MemoryRecord[];
  historicalContext: string;
  aiReasoning: string;
  businessAction: {
    actionType: string;
    details: string;
    status: 'Executed' | 'Pending Human Confirmation';
  };
  newMemoryCreated?: MemoryRecord;
  durationMs: number;
}
```

I think of this as an action boundary.

“The model generated this response” and “the system changed something in a business workflow” are different events. Keeping them separate gives me a place for policy checks, approval rules, audit logging, and confidence thresholds without making the generated response itself an irreversible command.

## A support interaction that actually benefits from memory

Consider the ERP example again.

A customer sends:

> “The ERP sync stopped after the quarterly credential rotation. We are in trading hours. What do I need to do?”

The agent recalls three useful pieces of information.

First, it recalls the previous resolution: the authentication token cache can become stale after credential rotation, and refreshing the token plus flushing Redis resolved the previous incident.

Second, it recalls the customer's environment: the ERP deployment, connector version, network arrangement, and known integrations.

Third, it recalls the customer's communication preference: they want direct technical steps rather than a generic support checklist.

Those memories change the response.

Instead of saying “check your credentials and restart the connector,” the agent can produce a targeted sequence based on the customer's known environment. It can also avoid a previously harmful action when that action is represented in memory as something that failed or created another problem.

After the interaction, the system can decide whether anything new is worth remembering. A confirmed connector behavior, a changed environment detail, or a newly verified resolution can become durable memory for the next incident.

The important part is what happens six months later. The next ticket does not begin with an empty context window.

## The architecture is deliberately replaceable

The repository has a useful seam between the application and its memory implementation.

The React application calls `/api` endpoints through a small service module. The Express server owns the API and model integration. Domain types define the contract between those layers.

That means the frontend should not care whether a memory came from an in-process store, a relational database, or Hindsight. It should receive a `MemoryRecord` and a trace.

That separation also makes it easier to change the model provider without redesigning the rest of the system.

I consider that more important than making the code look “agentic.” The system has ordinary application boundaries: API calls, typed objects, configuration, persistence, and business actions. The LLM sits inside those boundaries instead of becoming the architecture.

The same principle extends to the related accounts-payable workflow in the application. Vendor memories can represent recurring discrepancies, payment terms, previous exceptions, and approval decisions. The memory mechanism is shared, but the business semantics remain different.

That is the right level of reuse: share the memory infrastructure, not the meaning of every memory.

## What I learned building it

### 1. Memory needs semantics

“Store the conversation” is not a memory strategy.

I get much more useful behavior when memory explicitly represents previous solutions, known issues, environments, preferences, and failed approaches. Those categories make retrieval easier to reason about and make the resulting behavior more explainable.

### 2. Retrieval should be visible

If I cannot see which memories influenced an answer, debugging becomes speculation.

Returning `retrievedMemories` and a `PipelineTrace` makes the retrieval step inspectable. It also gives operators a way to understand why an automated action happened.

### 3. Long-term memory should not be the model

The model should reason over memory; it should not be the memory system.

Keeping Hindsight as a dedicated layer means I can change models without throwing away customer history or coupling persistence to a particular inference provider.

### 4. “What worked before” is different from documentation

Documentation tells me what should work.

Customer memory tells me what actually worked for this customer in this environment.

Those are different datasets. A support system benefits from having both.

### 5. Automation needs an action boundary

Generating text and changing the world are different operations.

The `PipelineTrace` model gives me a place to represent that distinction explicitly. An action can be executed or require human confirmation, which leaves room for policy, approvals, audit records, and operational controls.

## The part I would keep even if the model changed tomorrow

The interesting engineering problem here is not that an LLM can answer a support ticket. That part is increasingly ordinary.

The harder problem is building a support system that gets better from experience without becoming an opaque collection of stale context.

For me, that means four things have to remain explicit: durable memory, targeted retrieval, typed business semantics, and an observable path from input to memory to reasoning to action.

That is why Hindsight is a major architectural component rather than a feature hidden behind the chat screen.

If the model changes, the UI changes, or the inference provider changes, I still want the same basic behavior:

When a customer comes back with the same problem, I should not make them tell the entire story again.

I want the system to remember what happened, what worked, what failed, and what matters about that customer—and then use that history carefully enough that the second answer is genuinely more informed than the first.
