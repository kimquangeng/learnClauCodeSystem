# 🛠️ HANDS-ON EXERCISES
## Theo từng Chapter của Claude Code Architecture

---

## CHAPTER 5: AGENT LOOP — Bài tập thực hành chi tiết

### Prerequisites: Cần nắm trước

Trước khi làm bài tập Chapter 5, bạn cần hiểu:

```typescript
// 1. Async Generator là gì
async function* myGenerator() {
  yield 1
  yield 2
  yield 3
}

// 2. Dùng với for await
for await (const num of myGenerator()) {
  console.log(num) // 1, 2, 3
}

// 3. Backpressure: generator chỉ yield khi consumer sẵn sàng
// 4. Return value: giá trị trả về từ generator.return()
```

---

### 🎯 Exercise 1: Build Simple Agent Loop (1-2 ngày)

**Mục tiêu:** Hiểu cách loop hoạt động

```typescript
// File: exercises/ch05/simple-agent-loop.ts

// Bước 1: Define basic types
interface Message {
  role: 'user' | 'assistant'
  content: string
  toolCalls?: ToolCall[]
}

interface ToolCall {
  id: string
  name: string
  input: Record<string, any>
}

interface ToolResult {
  toolCallId: string
  result: string
  error?: string
}

type TerminalReason = 
  | { reason: 'completed' }
  | { reason: 'tool_calls' }
  | { reason: 'max_turns' }
  | { reason: 'error'; error: string }

// Bước 2: Viết function* đơn giản
function* simpleAgentLoop(
  messages: Message[],
  maxTurns: number = 10
): Generator<Message | ToolResult, TerminalReason, unknown> {
  
  let turnCount = 0
  
  while (turnCount < maxTurns) {
    // Gọi model giả lập
    const response = generateMockResponse(messages)
    
    yield { role: 'assistant', content: response.content }
    
    // Nếu không có tool calls -> done
    if (!response.toolCalls || response.toolCalls.length === 0) {
      return { reason: 'completed' }
    }
    
    // Execute tools
    for (const toolCall of response.toolCalls) {
      const result = executeTool(toolCall)
      yield result
      
      // Thêm vào messages
      messages.push(
        { role: 'assistant', content: '', toolCalls: [toolCall] },
        { role: 'user', content: result.result }
      )
    }
    
    turnCount++
  }
  
  return { reason: 'max_turns' }
}

// Bước 3: Chạy thử
const messages: Message[] = [
  { role: 'user', content: 'Read package.json' }
]

for await (const event of simpleAgentLoop(messages)) {
  console.log('Event:', event)
}
```

**Nhiệm vụ:**
1. Implement `generateMockResponse` giả lập model response
2. Implement `executeTool` đơn giản
3. Thêm error handling
4. Test với 3 turns

---

### 🎯 Exercise 2: Thêm State Management (1-2 ngày)

**Mục tiêu:** Hiểu immutable state transitions

```typescript
// File: exercises/ch05/state-management.ts

// Bước 1: Define state object
interface LoopState {
  messages: Message[]
  turnCount: number
  maxOutputTokensOverride: number | undefined
  hasAttemptedReactiveCompact: boolean
  transition: string | undefined
}

// Bước 2: Viết transition function
function createNextState(
  current: LoopState,
  updates: {
    messages?: Message[]
    turnCount?: number
    maxOutputTokensOverride?: number | undefined
    hasAttemptedReactiveCompact?: boolean
    transition: string
  }
): LoopState {
  // KEY POINT: Reconstruct entire state, không mutate
  return {
    messages: updates.messages ?? current.messages,
    turnCount: updates.turnCount ?? current.turnCount,
    maxOutputTokensOverride: updates.maxOutputTokensOverride,
    hasAttemptedReactiveCompact: updates.hasAttemptedReactiveCompact ?? false,
    transition: updates.transition
  }
}

// Bước 3: Sử dụng trong loop
async function* agentLoopWithState(initialMessages: Message[]) {
  let state: LoopState = {
    messages: initialMessages,
    turnCount: 0,
    maxOutputTokensOverride: undefined,
    hasAttemptedReactiveCompact: false,
    transition: undefined
  }
  
  while (true) {
    // Process...
    const response = await callModel(state.messages)
    
    if (!response.toolCalls) {
      return { reason: 'completed', state }
    }
    
    const toolResults = await executeTools(response.toolCalls)
    const newMessages = [
      ...state.messages,
      response,
      ...toolResults.map(r => ({ role: 'user' as const, content: r.result }))
    ]
    
    // KEY POINT: Recreate state
    state = createNextState(state, {
      messages: newMessages,
      turnCount: state.turnCount + 1,
      transition: 'next_turn'
    })
    
    yield { turnCount: state.turnCount, lastMessage: newMessages.at(-1) }
  }
}
```

**Nhiệm vụ:**
1. Thêm logging để track state changes
2. Verify immutable transitions (dùng console.log trước và sau)
3. Thêm `maxOutputTokensOverride` escalation

---

### 🎯 Exercise 3: Implement 4-Layer Compression (2-3 ngày)

**Mục tiêu:** Hiểu context management

```typescript
// File: exercises/ch05/context-compression.ts

interface CompressionResult {
  messages: Message[]
  freedTokens: number
  action: 'snip' | 'microcompact' | 'collapse' | 'autocompact' | 'none'
}

// Layer 0: Tool Result Budget
function applyToolResultBudget(messages: Message[], maxChars: number): Message[] {
  // Implement: cắt tool results > maxChars
}

// Layer 1: Snip Compact
function snipCompact(messages: Message[], thresholdTokens: number): CompressionResult {
  // Implement: xóa messages cũ nhất cho đến khi < threshold
  return {
    messages: [...],
    freedTokens: 5000,
    action: 'snip'
  }
}

// Layer 2: Microcompact  
function microcompact(messages: Message[], keepToolUseIds: Set<string>): CompressionResult {
  // Implement: xóa tool results không còn needed
  // Chỉ xóa messages mà KHÔNG phải tool_use messages
  return {
    messages: [...],
    freedTokens: 3000,
    action: 'microcompact'
  }
}

// Layer 3: Context Collapse
function contextCollapse(messages: Message[], model: any): CompressionResult {
  // Implement: thuê model summarize old messages
  return {
    messages: [
      { role: 'user', content: '[Earlier conversation summarized]' },
      ...messages.slice(-10) // keep recent messages
    ],
    freedTokens: 20000,
    action: 'collapse'
  }
}

// Layer 4: Auto-Compact (full summarization)
async function autoCompact(messages: Message[], model: any): Promise<CompressionResult> {
  // Implement: fork model để summarize toàn bộ history
}

// Full pipeline
async function compressMessages(messages: Message[], config: {
  model: any
  contextWindow: number
  maxOutput: number
}): Promise<Message[]> {
  
  let result = messages
  
  // Calculate thresholds
  const effectiveWindow = config.contextWindow - Math.min(config.maxOutput, 20000)
  const autoCompactThreshold = effectiveWindow - 13000 // 13K buffer
  
  // Layer 1: Snip (nếu token count > threshold)
  if (countTokens(result) > autoCompactThreshold) {
    result = snipCompact(result, autoCompactThreshold).messages
  }
  
  // Layer 2: Microcompact
  result = microcompact(result, new Set()).messages
  
  // Layer 3: Context Collapse
  if (countTokens(result) > autoCompactThreshold) {
    result = contextCollapse(result, null).messages
  }
  
  // Layer 4: Auto-Compact
  if (countTokens(result) > effectiveWindow - 3000) {
    result = await autoCompact(result, config.model).then(r => r.messages)
  }
  
  return result
}
```

**Nhiệm vụ:**
1. Implement mỗi layer với mock token counting
2. Test với message history 100 messages
3. Verify thứ tự: Layer 1 -> Layer 2 -> Layer 3 -> Layer 4
4. Thêm circuit breaker (sau 3 failures thì dừng)

---

### 🎯 Exercise 4: Error Recovery Ladder (1-2 ngày)

**Mục tiêu:** Hiểu cách xử lý errors theo layers

```typescript
// File: exercises/ch05/error-recovery.ts

type RecoveryAction =
  | { action: 'retry_with_collapse' }
  | { action: 'retry_with_compact' }
  | { action: 'escalate_tokens' }
  | { action: 'multi_turn_recovery'; attempt: number }
  | { action: 'surface_error' }

class ErrorRecovery {
  private consecutiveFailures = 0
  private maxOutputRecoveryAttempts = 0
  
  handleError(error: any, state: LoopState): RecoveryAction | null {
    // Error 1: Prompt too long (413)
    if (error.status === 413 || error.type === 'prompt_too_long') {
      this.consecutiveFailures++
      
      // Circuit breaker
      if (this.consecutiveFailures > 3) {
        return { action: 'surface_error' }
      }
      
      // Layer 1: Context collapse
      if (!state.hasAttemptedReactiveCompact) {
        return { action: 'retry_with_collapse' }
      }
      
      // Layer 2: Full compact
      return { action: 'retry_with_compact' }
    }
    
    // Error 2: Max output tokens hit
    if (error.type === 'max_output_tokens') {
      // Layer 1: Escalate 8K -> 64K
      if (!state.maxOutputTokensOverride) {
        return { action: 'escalate_tokens' }
      }
      
      // Layer 2: Multi-turn recovery
      this.maxOutputRecoveryAttempts++
      if (this.maxOutputRecoveryAttempts <= 3) {
        return { action: 'multi_turn_recovery'; attempt: this.maxOutputRecoveryAttempts }
      }
      
      return { action: 'surface_error' }
    }
    
    // Other errors: surface immediately
    return { action: 'surface_error' }
  }
  
  reset() {
    this.consecutiveFailures = 0
    this.maxOutputRecoveryAttempts = 0
  }
}

// Sử dụng trong loop
async function* agentLoopWithErrorRecovery(messages: Message[]) {
  let state = createInitialState(messages)
  const recovery = new ErrorRecovery()
  
  while (true) {
    try {
      const response = await callModel(state.messages)
      
      if (response.error) {
        const action = recovery.handleError(response.error, state)
        
        if (action?.action === 'surface_error') {
          return { reason: 'error', error: response.error }
        }
        
        if (action?.action === 'retry_with_compact') {
          state = createNextState(state, {
            hasAttemptedReactiveCompact: true,
            transition: 'reactive_compact_retry'
          })
          continue
        }
      }
      
      // Normal flow...
      
    } catch (error) {
      // Handle...
    }
  }
}
```

**Nhiệm vụ:**
1. Implement all error recovery paths
2. Add death spiral guard (consecutive failures > 3)
3. Test: 413 error -> collapse -> retry -> success
4. Test: 413 error -> collapse -> retry -> fail -> compact -> success

---

### 🎯 Exercise 5: Full Agent Loop với Streaming (2-3 ngày)

**Mục tiêu:** Implement speculative execution

```typescript
// File: exercises/ch05/full-agent-loop.ts

// Mock streaming model
async function* streamModel(messages: Message[]) {
  // Simulate streaming tokens
  const response = await generateMockResponse(messages)
  
  for (const token of response.content.split(' ')) {
    yield { type: 'token', data: token + ' ' }
    
    // Simulate tool_use arriving mid-stream
    if (response.toolCalls && token === response.toolCalls[0].name) {
      yield { type: 'tool_use', data: response.toolCalls[0] }
    }
  }
  
  yield { type: 'done', data: response }
}

// Streaming executor - KEY PATTERN
async function streamingExecutor(
  stream: AsyncGenerator<any>,
  tools: Map<string, Tool>,
  onToolResult: (result: ToolResult) => void
): Promise<Map<string, ToolResult>> {
  const results = new Map<string, ToolResult>()
  const pendingSafeTools: Promise<ToolResult>[] = []
  let responseComplete = false
  
  for await (const event of stream) {
    if (event.type === 'tool_use') {
      const tool = tools.get(event.data.name)
      if (!tool) continue
      
      // KEY: Start safe tools DURING streaming, not after
      if (tool.isConcurrencySafe(event.data.input)) {
        pendingSafeTools.push(executeTool(tool, event.data.input, event.data.id))
      }
    }
    
    if (event.type === 'done') {
      responseComplete = true
      
      // Execute remaining tools serially
      for (const tool of event.response.toolCalls ?? []) {
        if (!tools.get(tool.name)?.isConcurrencySafe(tool.input)) {
          const result = await executeTool(tools.get(tool.name)!, tool.input, tool.id)
          results.set(tool.id, result)
          onToolResult(result)
        }
      }
    }
  }
  
  // Wait for concurrent tools
  const concurrentResults = await Promise.all(pendingSafeTools)
  for (const result of concurrentResults) {
    results.set(result.toolCallId, result)
    onToolResult(result)
  }
  
  return results
}

// Full agent loop
async function* agentLoop(messages: Message[], tools: Map<string, Tool>) {
  let state = createInitialState(messages)
  
  while (true) {
    // Compress if needed
    if (shouldCompress(state.messages)) {
      state = compressState(state)
    }
    
    // Stream and execute
    const stream = streamModel(state.messages)
    const toolResults = await streamingExecutor(stream, tools, (result) => {
      state = appendToolResult(state, result)
    })
    
    // Update state with tool results
    state = createNextState(state, {
      messages: [...state.messages, ...toolResults.map(r => ({
        role: 'user' as const,
        content: r.result
      }))],
      turnCount: state.turnCount + 1,
      transition: 'next_turn'
    })
    
    // Check completion
    const lastResponse = toolResults[toolResults.length - 1]?.response
    if (!lastResponse?.toolCalls?.length) {
      return { reason: 'completed' }
    }
  }
}
```

**Nhiệm vụ:**
1. Implement mock streaming với tool_use arrivals
2. Verify: safe tools start BEFORE response complete
3. Verify: write tools wait until response complete
4. Test race condition: tool done vs model done

---

## CHAPTER 7: CONCURRENCY — Bài tập thực hành chi tiết

### 🎯 Exercise 1: Partition Algorithm (1-2 ngày)

**Mục tiêu:** Hiểu cách chia tools thành concurrent/serial batches

```typescript
// File: exercises/ch07/partition.ts

interface Tool {
  name: string
  isConcurrencySafe(input: any): boolean
  isReadOnly(input: any): boolean
}

interface ToolCall {
  id: string
  name: string
  input: any
}

// Partition function - chia tools thành batches
interface Partition {
  concurrent: ToolCall[]
  serial: ToolCall[]
}

function partitionTools(
  toolCalls: ToolCall[],
  toolRegistry: Map<string, Tool>
): Partition {
  const concurrent: ToolCall[] = []
  const serial: ToolCall[] = []
  
  for (const toolCall of toolCalls) {
    const tool = toolRegistry.get(toolCall.name)
    if (!tool) {
      // Unknown tool -> serial (fail-closed)
      serial.push(toolCall)
      continue
    }
    
    // Tool phải vừa concurrency-safe VỪA read-only
    // BashTool: "ls" = safe, "rm file" = not safe
    const isSafe = tool.isConcurrencySafe(toolCall.input)
    const isReadOnly = tool.isReadOnly(toolCall.input)
    
    if (isSafe && isReadOnly) {
      concurrent.push(toolCall)
    } else {
      serial.push(toolCall)
    }
  }
  
  return { concurrent, serial }
}

// Execution: run concurrent trước, rồi serial
async function executePartitioned(
  partition: Partition,
  toolRegistry: Map<string, Tool>,
  onProgress: (result: ToolResult) => void
): Promise<ToolResult[]> {
  const results: ToolResult[] = []
  
  // Step 1: Run all concurrent tools in parallel
  const concurrentPromises = partition.concurrent.map(toolCall => 
    executeTool(toolRegistry.get(toolCall.name)!, toolCall.input, toolCall.id)
      .then(result => {
        onProgress(result)
        return result
      })
  )
  
  const concurrentResults = await Promise.all(concurrentPromises)
  results.push(...concurrentResults)
  
  // Step 2: Run serial tools one by one
  for (const toolCall of partition.serial) {
    const tool = toolRegistry.get(toolCall.name)!
    const result = await executeTool(tool, toolCall.input, toolCall.id)
    onProgress(result)
    results.push(result)
  }
  
  return results
}
```

**Nhiệm vụ:**
1. Thêm logging để verify execution order
2. Test với 5 tool calls: 3 read + 2 write
3. Verify: concurrent tools start cùng lúc
4. Verify: serial tools run sequentially

---

### 🎯 Exercise 2: Speculative Execution (2-3 ngày)

**Mục tiêu:** Implement pattern QUAN TRỌNG NHẤT của Claude Code

```typescript
// File: exercises/ch07/speculative-execution.ts

// Speculative executor: bắt đầu safe tools NGAY khi detect tool_use
// KHÔNG đợi model response complete

class SpeculativeExecutor {
  private pendingResults: Map<string, Promise<ToolResult>> = new Map()
  private completedResults: Map<string, ToolResult> = new Map()
  private toolRegistry: Map<string, Tool>
  
  constructor(toolRegistry: Map<string, Tool>) {
    this.toolRegistry = toolRegistry
  }
  
  // Gọi khi detect tool_use block trong stream
  onToolUseDetected(toolCall: ToolCall) {
    const tool = this.toolRegistry.get(toolCall.name)
    if (!tool) return
    
    // Chỉ start safe tools
    if (tool.isConcurrencySafe(toolCall.input) && tool.isReadOnly(toolCall.input)) {
      // KEY: Start execution NGAY, không đợi response complete
      const promise = executeTool(tool, toolCall.input, toolCall.id)
      this.pendingResults.set(toolCall.id, promise)
      
      // Track completion
      promise.then(result => {
        this.pendingResults.delete(toolCall.id)
        this.completedResults.set(toolCall.id, result)
      })
    }
  }
  
  // Gọi khi response complete
  async onResponseComplete(toolCalls: ToolCall[]): Promise<ToolResult[]> {
    const results: ToolResult[] = []
    
    for (const toolCall of toolCalls) {
      // 1. Check if already completed speculatively
      if (this.completedResults.has(toolCall.id)) {
        results.push(this.completedResults.get(toolCall.id)!)
        continue
      }
      
      // 2. Check if still pending
      if (this.pendingResults.has(toolCall.id)) {
        const result = await this.pendingResults.get(toolCall.id)
        results.push(result)
        continue
      }
      
      // 3. New tool - execute now
      const tool = this.toolRegistry.get(toolCall.name)
      if (tool) {
        const result = await executeTool(tool, toolCall.input, toolCall.id)
        results.push(result)
      }
    }
    
    return results
  }
  
  // Drain all pending when abort
  async drainAll(): Promise<ToolResult[]> {
    const results: ToolResult[] = []
    for (const [id, promise] of this.pendingResults) {
      try {
        const result = await Promise.race([
          promise,
          new Promise((_, reject) => 
            setTimeout(() => reject(new Error('Timeout')), 5000)
          )
        ])
        results.push(result)
      } catch {
        // Create error result
        results.push({ toolCallId: id, result: '', error: 'Aborted' })
      }
    }
    this.pendingResults.clear()
    return results
  }
}

// Usage in agent loop
async function* agentLoopWithSpeculativeExecution(messages: Message[]) {
  const executor = new SpeculativeExecutor(toolRegistry)
  
  // Stream model response
  const stream = streamModel(messages)
  
  for await (const event of stream) {
    if (event.type === 'tool_use') {
      // KEY: Bắt đầu execution NGAY
      executor.onToolUseDetected(event.data)
    }
    
    if (event.type === 'done') {
      // Đợi remaining tools (bao gồm cả speculative đã complete)
      const results = await executor.onResponseComplete(event.data.toolCalls)
      
      // Add results to messages
      messages.push(...results.map(r => ({
        role: 'user' as const,
        content: r.result
      })))
    }
  }
}
```

**Nhiệm vụ:**
1. Implement mock streaming với mid-stream tool_use
2. Measure time: model response complete vs tool execution complete
3. Verify: Read tool complete TRƯỚC model done
4. Test race condition handling

---

### 🎯 Exercise 3: Full Streaming Tool Executor (2-3 ngày)

```typescript
// File: exercises/ch07/streaming-executor.ts

class StreamingToolExecutor {
  private toolRegistry: Map<string, Tool>
  private activeTools: Map<string, Promise<ToolResult>> = new Map()
  
  constructor(toolRegistry: Map<string, Tool>) {
    this.toolRegistry = toolRegistry
  }
  
  // Parse tool calls from streaming response
  extractToolCalls(delta: string): ToolCall[] {
    // Mock: parse JSON-like tool calls from stream
    const matches = delta.matchAll(/tool_use\("(\w+)", ({.*?})\)/g)
    return Array.from(matches).map(m => ({
      id: generateId(),
      name: m[1],
      input: JSON.parse(m[2])
    }))
  }
  
  // Start tool execution during streaming
  async startTool(
    toolCall: ToolCall, 
    signal?: AbortSignal
  ): Promise<ToolResult> {
    const tool = this.toolRegistry.get(toolCall.name)
    if (!tool) {
      return { toolCallId: toolCall.id, error: 'Unknown tool', result: '' }
    }
    
    // Run with abort support
    const promise = executeTool(tool, toolCall.input, toolCall.id)
    
    if (signal?.aborted) {
      return { toolCallId: toolCall.id, error: 'Aborted', result: '' }
    }
    
    this.activeTools.set(toolCall.id, promise)
    
    try {
      return await promise
    } finally {
      this.activeTools.delete(toolCall.id)
    }
  }
  
  // Drain all active tools (on abort)
  async drain(signal: AbortSignal): Promise<ToolResult[]> {
    const results: ToolResult[] = []
    
    for (const [id, promise] of this.activeTools) {
      // Race with abort signal
      const result = await Promise.race([
        promise,
        new Promise<ToolResult>(resolve => {
          signal.addEventListener('abort', () => {
            resolve({ toolCallId: id, error: 'Aborted', result: '' })
          })
        })
      ])
      results.push(result)
    }
    
    this.activeTools.clear()
    return results
  }
}

// Full streaming flow
async function* streamingAgentLoop(messages: Message[]) {
  const executor = new StreamingToolExecutor(toolRegistry)
  const abortController = new AbortController()
  
  try {
    const stream = streamModelWithAbort(messages, abortController.signal)
    
    let currentToolCalls: ToolCall[] = []
    
    for await (const event of stream) {
      if (event.type === 'delta') {
        // Extract tool calls from streaming delta
        const newToolCalls = executor.extractToolCalls(event.data)
        
        for (const toolCall of newToolCalls) {
          const tool = toolRegistry.get(toolCall.name)!
          
          // Speculative execution for safe tools
          if (tool.isConcurrencySafe(toolCall.input) && tool.isReadOnly(toolCall.input)) {
            executor.startTool(toolCall)
          } else {
            currentToolCalls.push(toolCall)
          }
        }
      }
      
      if (event.type === 'done') {
        // Wait for all tools to complete
        const results = await executor.onResponseComplete(currentToolCalls)
        
        messages.push(...results.map(r => ({
          role: 'user' as const,
          content: r.result
        })))
        
        currentToolCalls = []
      }
    }
  } finally {
    // Cleanup on abort
    await executor.drain(abortController.signal)
  }
}
```

**Nhiệm vụ:**
1. Implement với mock streaming
2. Verify speculative execution timing
3. Handle abort properly
4. Test concurrent vs serial distinction

---

## CHAPTER 3: STATE — Bài tập thực hành chi tiết

### 🎯 Exercise 1: Two-Tier State Architecture (1-2 ngày)

```typescript
// File: exercises/ch03/two-tier-state.ts

// ============== TIER 1: Bootstrap State (Mutable Singleton) ==============
// Đây là process-level state, KHÔNG trigger re-renders

interface BootstrapState {
  sessionId: string
  projectRoot: string
  currentModel: string
  permissionMode: 'default' | 'plan' | 'bypass' | 'auto'
  costTracker: {
    totalCost: number
    inputTokens: number
    outputTokens: number
  }
  flags: {
    afkModeHeaderLatched: boolean | null  // Sticky latch!
    thinkingClearLatched: boolean | null   // Sticky latch!
  }
}

// Singleton - chỉ tạo 1 lần
const STATE: BootstrapState = {
  sessionId: generateSessionId(),
  projectRoot: process.cwd(),
  currentModel: 'claude-3-5-sonnet-20241022',
  permissionMode: 'default',
  costTracker: { totalCost: 0, inputTokens: 0, outputTokens: 0 },
  flags: { afkModeHeaderLatched: null, thinkingClearLatched: null }
}

// Getters/Setters - encapsulate access
export function getSessionId(): string {
  return STATE.sessionId
}

export function setCurrentModel(model: string): void {
  STATE.currentModel = model
}

export function getCurrentModel(): string {
  return STATE.currentModel
}

// Sticky Latch pattern
export function setAfkModeHeaderLatched(value: boolean): void {
  if (STATE.flags.afkModeHeaderLatched === null) {
    STATE.flags.afkModeHeaderLatched = value
  }
  // Once set, NEVER unset!
}

export function shouldSendAfkHeader(currentlyActive: boolean): boolean {
  const latched = STATE.flags.afkModeHeaderLatched
  if (latched === true) return true
  if (currentlyActive) {
    setAfkModeHeaderLatched(true)
    return true
  }
  return false
}

// ============== TIER 2: AppState (Reactive Store) ==============
// Đây là UI state, trigger re-renders

interface AppState {
  messages: Message[]
  inputMode: 'editing' | 'waiting'
  progress: number
  toolResults: Map<string, ToolResult>
}

// Simple reactive store (Zustand-like, 34 lines)
function createStore<T>(initial: T, onChange?: (prev: T, next: T) => void) {
  let current = initial
  const subscribers = new Set<(state: T) => void>()
  
  return {
    getState: () => current,
    
    setState: (updater: T | ((prev: T) => T)) => {
      const next = typeof updater === 'function' 
        ? (updater as Function)(current) 
        : updater
      
      // Object.is check - prevent unnecessary re-renders
      if (Object.is(next, current)) return
      
      const prev = current
      current = next
      
      // onChange fires BEFORE subscribers
      onChange?.(prev, next)
      
      // Notify subscribers
      subscribers.forEach(cb => cb(current))
    },
    
    subscribe: (cb: (state: T) => void) => {
      subscribers.add(cb)
      return () => subscribers.delete(cb)
    }
  }
}

// Create store
const appStore = createStore<AppState>(
  {
    messages: [],
    inputMode: 'editing',
    progress: 0,
    toolResults: new Map()
  },
  (prev, next) => {
    // Side effects: sync with Bootstrap State
    // Khi model change trong UI, sync xuống Bootstrap
    if (prev.currentModel !== next.currentModel) {
      setCurrentModel(next.currentModel as any)
    }
  }
)

// React integration
function useAppState<T>(selector: (state: AppState) => T): T {
  // Simplified - actual implementation uses useSyncExternalStore
  let currentValue = selector(appStore.getState())
  
  appStore.subscribe(state => {
    const newValue = selector(state)
    if (!Object.is(newValue, currentValue)) {
      currentValue = newValue
      // Trigger re-render
    }
  })
  
  return currentValue
}
```

**Nhiệm vụ:**
1. Implement cả 2 tiers
2. Demo sticky latch: toggle flag, verify never unset
3. Demo onChange sync: change model in store, verify STATE updated
4. Test React component re-renders

---

## CHAPTER 6: TOOLS — Bài tập thực hành chi tiết

### 🎯 Exercise 1: Build Tool Interface (1-2 ngày)

```typescript
// File: exercises/ch06/tool-interface.ts

import { z } from 'zod'

// 1. Define tool interface
interface Tool<I, O, P> {
  name: string
  description: string
  inputSchema: z.ZodType<I>
  
  // Core methods
  call(input: I): Promise<ToolResult<O>>
  
  // Safety methods (input-dependent!)
  isConcurrencySafe(input: I): boolean
  isReadOnly(input: I): boolean
  
  // Permissions
  checkPermissions(input: I): PermissionDecision
  
  // UI
  render(input: I, result: O): string
}

// 2. BuildTool factory với fail-closed defaults
const SAFE_DEFAULTS = {
  isConcurrencySafe: () => false,  // Fail-closed: run serially by default
  isReadOnly: () => false,         // Fail-closed: treat as write by default
  checkPermissions: (input: any) => ({ behavior: 'allow' }),
}

function buildTool<T extends Tool<any, any, any>>(definition: T): T {
  return {
    ...SAFE_DEFAULTS,
    ...definition
  }
}

// 3. Example: BashTool
const BashTool = buildTool({
  name: 'Bash',
  description: 'Execute shell commands',
  inputSchema: z.object({
    command: z.string()
  }),
  
  isConcurrencySafe(input): boolean {
    return this.isReadOnly(input)
  },
  
  isReadOnly(input): boolean {
    const cmd = input.command
    // Parse command and check safety
    return isReadOnlyCommand(cmd)  // "ls", "cat", "grep" = safe
                                  // "rm", "mv", ">" = not safe
  },
  
  async call(input) {
    const result = await exec(input.command)
    return { success: true, output: result.stdout }
  }
})

// 4. Execute pipeline (14 steps simplified)
async function executeTool(
  toolCall: { name: string, input: any },
  toolRegistry: Map<string, Tool<any, any, any>>,
  permissionChecker: (tool: string) => boolean
) {
  // Step 1: Lookup tool
  const tool = toolRegistry.get(toolCall.name)
  if (!tool) throw new Error(`Unknown tool: ${toolCall.name}`)
  
  // Step 2: Zod validation
  const parsed = tool.inputSchema.safeParse(toolCall.input)
  if (!parsed.success) throw new Error('Invalid input')
  
  // Step 3: Permission check
  const decision = tool.checkPermissions(parsed.data)
  if (decision.behavior === 'deny') {
    throw new Error('Permission denied')
  }
  
  // Step 4: Global permission
  if (!permissionChecker(toolCall.name)) {
    throw new Error('Permission denied by global checker')
  }
  
  // Step 5: Execute
  return await tool.call(parsed.data)
}
```

**Nhiệm vụ:**
1. Implement BashTool, ReadTool, WriteTool
2. Test fail-closed: tool không implement isReadOnly -> false
3. Test permission flow
4. Implement tool result budgeting

---

## PROJECT: Mini Claude Code Clone

### Overview

Sau khi hoàn thành các exercises trên, xây một CLI agent hoàn chỉnh:

```typescript
// File: mini-claude-cli/index.ts

// Features:
// 1. Async generator agent loop
// 2. 5 basic tools: Read, Write, Edit, Bash, Grep
// 3. Streaming tool execution
// 4. Two-tier state (Bootstrap + AppState)
// 5. Basic permission system (plan mode)
// 6. Context compression (2 layers)
// 7. Error recovery

// Structure:
/mini-claude-cli
  /src
    /core
      agent-loop.ts       // Async generator loop
      state.ts            // Two-tier state
      context.ts          // Compression layers
      errors.ts           // Error recovery
    /tools
      tool-registry.ts    // Tool registry
      bash-tool.ts
      read-tool.ts
      write-tool.ts
      edit-tool.ts
      grep-tool.ts
    /api
      client.ts          // API client
      streaming.ts       // Streaming utilities
    /ui
      repl.ts            // Terminal UI
    index.ts
  package.json
  tsconfig.json
```

---

## CHECKLIST HOÀN THÀNH

### Chapter 5 Exercises:
- [ ] Exercise 1: Simple agent loop
- [ ] Exercise 2: State management
- [ ] Exercise 3: 4-layer compression
- [ ] Exercise 4: Error recovery ladder
- [ ] Exercise 5: Full agent loop với streaming

### Chapter 7 Exercises:
- [ ] Exercise 1: Partition algorithm
- [ ] Exercise 2: Speculative execution
- [ ] Exercise 3: Full streaming executor

### Chapter 3 Exercises:
- [ ] Exercise 1: Two-tier state architecture

### Chapter 6 Exercises:
- [ ] Exercise 1: Build tool interface

### Final Project:
- [ ] Mini Claude Code Clone