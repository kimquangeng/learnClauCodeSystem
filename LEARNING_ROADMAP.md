# 🚀 LỘ TRÌNH HỌC TẬP: CLAUDE CODE ARCHITECTURE
## Dành cho Entry-Level Engineer

---

## PHẦN 0: PREREQUISITES (Cần ôn lại trước)

### Week 0: Nền tảng cần có

**Trước khi đọc bất kỳ chapter nào, bạn cần nắm vững:**

### 1. TypeScript / JavaScript hiện đại
- [ ] Async/Await và Promise
- [ ] Generators (function*, yield)
- [ ] Async Iterators và Async Generators
- [ ] Decorators (nếu có)
- [ ] Generics
- [ ] Union Types và Discriminated Unions

**Tài liệu để học:**
- MDN: Async Iterators https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for-await...of
- TypeScript Handbook: Generics

### 2. Design Patterns cơ bản
- [ ] State Machine
- [ ] Singleton
- [ ] Factory Pattern
- [ ] Dependency Injection
- [ ] Observer / Pub-Sub

**Tài liệu để học:**
- Head First Design Patterns (quyển cũ nhưng vẫn tốt)

### 3. Khái niệm AI/LLM cơ bản
- [ ] Transformer architecture (ở mức high-level)
- [ ] Tokens là gì? Context window là gì?
- [ ] Streaming response là gì?
- [ ] Tool use / Function calling là gì?

**Tài liệu để học:**
- OpenAI: https://platform.openai.com/docs/guides/text-generation
- Anthropic: https://docs.anthropic.com/

---

## PHẦN 1: HIGH-LEVEL OVERVIEW (1-2 tuần)

### Mục tiêu: Hiểu TỔNG QUAN bằng cách không đọc code

### Bước 1: Đọc lướt CHƯƠNG 1 (Architecture Overview)

**Lần 1 (30 phút):** Đọc không cần hiểu hết
- Đọc toàn bộ ch01-architecture.md
- Highlight 6 abstraction KEY
- Vẽ lại diagram bằng tay 3 lần

**Lần 2 (1 tiếng):** Đọc với giấy bút
- Ghi ra 10 câu hỏi về những chỗ chưa hiểu
- Copy diagram vào notebook

**Lần 3 (2 tiếng):** Research những chỗ chưa hiểu
- Google từng khái niệm
- Xem YouTube về Async Generator, State Machine

### 📌 **Checkpoint cho Phần 1:**
```
Bạn có thể giải thích bằng lời:
1. AI Agent khác CLI thường chỗ nào?
2. 6 abstraction chính là gì? (không cần chi tiết)
3. User -> Model -> Tools -> Model: flow cơ bản ra sao?
4. Permission system có 7 modes, bạn nhớ được mấy?
```

---

## PHẦN 2: BUILD TỪ GROUND UP (3-4 tuần)

### Mục tiêu: Tự xây một cái đơn giản trước

### Bước 1: Xây Simple Agent Loop

**Thực hành 1: Async Generator Agent Loop**

```typescript
// Bài tập: Xây agent loop đơn giản nhất có thể
// Chỉ cần: nhận input -> gọi model -> execute tools -> loop

// Step 1: Viết function* đơn giản (KHÔNG cần async)
function* simpleAgentLoop(messages: string[]) {
  // pseudocode
  while (true) {
    const response = generateResponse(messages)
    if (!response.toolCalls) break
    const results = executeTools(response.toolCalls)
    messages.push(response, ...results)
  }
  return { reason: 'completed' }
}

// Step 2: Biến thành async generator
async function* agentLoop(params) {
  // thêm streaming
}

// Step 3: Thêm state transitions
```

**Bài tập cụ thể:**

Ngày 1-2:
```
Tạo file: my-first-agent.ts
Xây:
1. Một interface cho Message
2. Một function parseAssistantResponse trả về tool calls
3. Một function executeReadTool() đơn giản
4. Một generator loop 10-20 lines thật sự đơn giản
```

Ngày 3-4:
```
Mở rộng:
1. Thêm state object với turnCount
2. Thêm error recovery đơn giản
3. Thêm một compression layer
```

### Bước 2: Xây Tool System đơn giản

**Tạo file: tool-system.ts**

```typescript
// 1. Define tool interface (như trong sách nhưng đơn giản hơn)
interface Tool<I, O, P> {
  name: string
  description: string
  inputSchema: any
  execute(input: I): Promise<O>
  concurrency: 'safe' | 'unsafe'  // điều này QUAN TRỌNG
  permission: 'read' | 'write' | 'dangerous'
}

// 2. Xây tool registry
class ToolRegistry {
  private tools: Map<string, Tool> = new Map()
  
  register(tool: Tool) { this.tools.set(tool.name, tool) }
  get(name: string): Tool | undefined { return this.tools.get(name) }
  list(): Tool[] { return Array.from(this.tools.values()) }
}

// 3. Simple execution pipeline
async function executeTool(tool: Tool, input: any) {
  // 1. Validate input (Zod)
  // 2. Check permission
  // 3. Execute
  // 4. Return result
}
```

**Bài tập:**
```
1. Tạo ReadTool, WriteTool, EditTool, BashTool
2. Mỗi tool tự khai báo concurrency và permission
3. Xây partition function: chia tools thành concurrent và serial
4. Xây streaming executor đơn giản (bắt đầu safe tools NGAY khi detect)
```

### Bước 3: Xây State Layer

```typescript
// Hai-tier state như trong sách

// Tier 1: Infrastructure state (singleton, mutable)
const STATE = {
  sessionId: generateId(),
  workingDirectory: process.cwd(),
  model: 'claude-3-5-sonnet',
  permissionMode: 'default',
  cost: { spent: 0, tokens: 0 },
  // ... 80 fields như trong sách nhưng BẮT ĐẦU với 10 fields thôi
}

// Tier 2: Reactive state (Zustand-shape)
interface AppState {
  messages: Message[]
  inputMode: 'editing' | 'waiting'
  progress: number
}
```

### 📌 **Checkpoint cho Phần 2:**
```
1. Chạy được agent loop của mình
2. Có thể execute 3 tools: Read, Write, Bash
3. State phân tách 2 tiers
4. Partition tools đúng: read=concurrent, write=serial
5. Stream tool execution HOẠT ĐỘNG
```

---

## PHẦN 3: ĐỌC SÁCH THEO THỨ TỰ (4-8 tuần)

### Mục tiêu: Mỗi tuần đọc 1-2 chương, với hands-on practice

### Tuần 1: Bootstrap & State

**Day 1-2: Đọc CHƯƠNG 2 (Bootstrap)**
```
High-level concepts:
- 5-phase init pipeline
- Module-level I/O parallelism
- Trust boundary

Thực hành:
- Xây init() function với fake delays
- Xem log output: "Phase 1... Phase 2..."
- Understand: tại sao bootstrap cần 5 phases?
```

**Day 3-5: Đọc CHƯƠNG 3 (State)**
```
High-level concepts:
- Bootstrap singleton (STATE)
- AppState reactive store
- Sticky latches
- Cost tracking

Thực hành:
- Mở rộng STATE object của mình
- Thêm cost tracking: đếm tokens, tính cost
- Thêm sticky latch: một khi set rồi thì KHÔNG unset
```

**Week 1 Checkpoint:**
```
✓ Init pipeline của mình chạy 5 phases
✓ STATE singleton tồn tại
✓ Cost tracking hoạt động
✓ Sticky latch có thể demo được
```

---

### Tuần 2: API Layer & Agent Loop

**Day 1-3: Đọc CHƯƠNG 4 (API Layer)**
```
High-level concepts:
- Multi-provider client (Direct, Bedrock, Vertex, Azure)
- Prompt caching
- Streaming
- Error recovery

Thực hành:
- Xây simple API client (dùng OpenAI-compatible API)
- Implement streaming với fetch
- Xử lý retry với exponential backoff
```

**Day 4-7: Đọc CHƯƠNG 5 (Agent Loop) - CONCENTRATED**
```
ĐÂY LÀ CHAPTER QUAN TRỌNG NHẤT - ĐỌC 3 LẦN

Lần 1 (2 giờ): Đọc toàn bộ
- Highlight mọi từ KHÔNG hiểu
- Vẽ state diagram bằng tay

Lần 2 (3 giờ): Đọc từng phần với code
- Tái tạo từng phần trong project của mình
- Phần nào không hiểu, google thêm

Lần 3 (2 giờ): Đọc lại toàn bộ
- Giờ phải hiểu 80% rồi
- Ghi ra 20% còn lại để hỏi hoặc research thêm
```

**Chi tiết cách đọc CHƯƠNG 5:**

```markdown
## Cấu trúc đọc Chapter 5 (7 ngày)

### Day 1: "Why an Async Generator"
- Hiểu: tại sao dùng generator thay vì callback/event emitter
- Thực hành: viết function* đơn giản, rồi convert sang async function*
- Ghi chú: backpressure, return value semantics, yield*

### Day 2: "What Callers Provide" + "Two-Layer Entry Point"
- Hiểu: LoopParams là gì? querySource phân biệt gì?
- Thực hành: định nghĩa LoopParams của mình

### Day 3: "The State Object" + "Immutable Transitions"
- Hiểu: state object có 10 fields, tại sao mỗi field tồn tại
- Thực hành: xây state object, thực hành immutable transitions
- QUAN TRỌNG: Hiểu tại sao PHẢI reconstruct state thay vì mutate

### Day 4: "The Loop Body" (Mermaid diagram)
- Vẽ lại diagram 5 lần
- Trace một request đi qua từng state
- Hiểu: ContextPipeline -> ModelStreaming -> PostStream -> ToolExecution

### Day 5: "Context Management: Four Compression Layers"
- Hiểu 4 layers: Tool Result Budget -> Snip -> Microcompact -> Auto-compact
- Thực hành: implement chỉ Layer 1 (Snip) trước
- Đọc kỹ phần thresholds và circuit breaker

### Day 6: "Error Recovery: The Escalation Ladder"
- Hiểu: có nhiều layers của error recovery
- Thực hành: viết try-catch với fallback
- QUAN TRỌNG: Hiểu death spiral guard (consecutive failures)

### Day 7: "Worked Example" + "Apply This"
- Đọc worked example: "Fix the bug in auth.ts"
- Trace 3 iterations
- Đọc Apply This: đây là 5 patterns bạn CẦN NHỚ
```

### 📌 **Checkpoint cho Tuần 2:**
```
✓ Hiểu async generator
✓ LoopParams có thể định nghĩa được
✓ State object có 10 fields
✓ Immutable transitions hoạt động
✓ 4 compression layers hiểu được thứ tự
✓ Error recovery ladder vẽ được
✓ Worked example trace được
```

---

### Tuần 3: Tools & Concurrency

**Day 1-3: Đọc CHƯƠNG 6 (Tools)**
```
High-level concepts:
- Tool interface (14-step pipeline)
- Permission system integration
- Tool schema và validation
- Rendering

Thực hành:
- Thêm tools mới vào tool system của mình
- Xây Zod validation cho mỗi tool
- Thử implement tool với progress reporting
```

**Day 4-7: Đọc CHƯƠNG 7 (Concurrency) - CONCENTRATED**
```
ĐÂY LÀ CHAPTER KHÓ NHẤT cho entry-level

Key concepts:
1. Partition algorithm
2. Streaming executor (speculative execution)
3. Batching by safety

Thực hành từng bước:
1. Viết partition function đơn giản
2. Xây sequential executor TRƯỚC
3. Thêm streaming executor
4. Implement speculative execution (start read tools NGAY)
```

**Chi tiết đọc Chapter 7:**

```markdown
## Cách đọc Chapter 7 (7 ngày)

### Lý thuyết cần nắm TRƯỚC KHI đọc:
1. Concurrent vs Parallel trong JS
2. Promise.all() vs sequential await
3. Event loop và queueing

### Day 1-2: "Speculative Execution" concept
- Hiểu: model còn đang stream, mình đã start read tools
- Đây là pattern QUAN TRỌNG NHẤT của Claude Code
- Thực hành: viết demo với setTimeout để simulate streaming

### Day 3-4: "Partition Algorithm"
- Đọc kỹ phần partition theo safety
- Thực hành: viết partition function
  - Read tools = concurrent (Promise.all)
  - Write tools = serial (sequential)

### Day 5-6: "Streaming Executor"
- Hiểu: executor bắt đầu khi detect tool_use block
- Không đợi model finish
- Handle race condition: tool done trước hay model done trước?

### Day 7: Implement đầy đủ
- Ghép tất cả lại
- Test với 5 tools: 3 read + 2 write
- Verify: write tools KHÔNG start cho đến model done
```

### 📌 **Checkpoint cho Tuần 3:**
```
✓ Tool interface 14 steps
✓ Partition algorithm hiểu được
✓ Speculative execution hoạt động
✓ Streaming executor xử lý được race condition
```

---

### Tuần 4: Sub-Agents & Fork Agents

**Day 1-4: Đọc CHƯƠNG 8 (Sub-Agents)**
```
High-level concepts:
- AgentTool spawns new query loop
- 15-step runAgent lifecycle
- Agent types (compact, session_memory, etc.)
- Permission bubble up

Đây là chapter DÀI NHẤT (58KB) - cần thời gian
```

**Day 5-7: Đọc CHƯƠNG 9 (Fork Agents)**
```
High-level concepts:
- Byte-identical prefix trick
- Cache sharing
- Cost optimization
```

### 📌 **Checkpoint cho Tuần 4:**
```
✓ Hiểu sub-agent là recursive query loop
✓ Permission bubble mode hoạt động
✓ Byte-identical prefix tiết kiệm tokens
```

---

### Tuần 5-6: Memory, Hooks, UI

**Day 1-3: Đọc CHƯƠNG 11 (Memory)**
```
- File-based memory
- 4-type taxonomy (project, user, team, session)
- LLM-powered relevance selection
- Staleness management
```

**Day 4-5: Đọc CHƯƠNG 12 (Extensibility/Skills)**
```
- Two-phase skill loading
- Lifecycle hooks
- Snapshot security
```

**Day 6-7: Đọc CHƯƠNG 13 (Terminal UI)**
```
- Ink/React custom renderer
- Double-buffer rendering
- Pool management
```

---

### Tuần 7: MCP, Remote, Performance

**Day 1-2: CHƯƠNG 15 (MCP)**
```
- 8 transports
- OAuth
- Tool wrapping
```

**Day 3-4: CHƯƠNG 16 (Remote)**
```
- Bridge v1/v2
- CCR
- Upstream proxy
```

**Day 5-7: CHƯƠNG 17 (Performance)**
```
- Startup optimization
- Prompt cache
- Search optimization
```

---

## PHẦN 4: DEEP DIVE & PROJECTS (Tuần 8+)

### Mục tiêu: Xây project thực tế để KHẮC SÂU kiến thức

### Project 1: Mini Claude Code Clone (4 tuần)

```
Xây một CLI agent đơn giản với:
- Async generator agent loop
- 5 basic tools (Read, Write, Edit, Bash, Grep)
- Streaming tool execution
- Two-tier state
- Basic permission system
- Context compression (2 layers)
- Error recovery

Đây là project TỐT NHẤT để học. Không cần hoàn hảo,
chỉ cần chạy được và HIỂU tại sao mỗi phần tồn tại.
```

### Project 2: Tool Registry & MCP Server (2 tuần)

```
Mở rộng project 1:
- Xây MCP server đơn giản
- Connect với Claude Desktop
- Test 3-5 tools qua MCP
```

### Project 3: Memory System (2 tuần)

```
Xây simple memory system:
- CLAUDE.md parsing
- LLM-powered relevance selection
- Staleness tracking
```

---

## PHẦN 5: TÀI LIỆU BỔ SUNG

### Khi đọc từng chapter, tham khảo thêm:

**Cho API Layer (Chapter 4):**
- https://docs.anthropic.com/claude/reference
- https://github.com/anthropics/anthropic-sdk-typescript

**Cho Tool System (Chapter 6):**
- https://json-schema.org/
- https://zod.dev/

**Cho Concurrency (Chapter 7):**
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all
- Viết demo: Promise.race() vs Promise.all()

**Cho State (Chapter 3):**
- https://github.com/pmndrs/zustand

**Cho UI (Chapter 13):**
- https://inkjs.org/ (nếu muốn hiểu Ink)
- React state management

---

## BẢNG THEO DÕI TIẾN ĐỘ

| Phase | Nội dung | Thời gian | Done? |
|-------|---------|-----------|-------|
| 0 | Prerequisites (TS, Patterns, AI) | 1 tuần | ☐ |
| 1 | High-level overview | 1-2 tuần | ☐ |
| 2 | Build from ground up | 3-4 tuần | ☐ |
| 3 | Đọc sách theo thứ tự | 8 tuần | ☐ |
| 4 | Deep dive & projects | Tuần 8+ | ☐ |

### Chi tiết Phase 3 (Đọc sách):

| Tuần | Chapters | Notes |
|------|----------|-------|
| 1 | Ch2 + Ch3 | Bootstrap & State |
| 2 | Ch4 + Ch5 | API Layer & Agent Loop ⭐ |
| 3 | Ch6 + Ch7 | Tools & Concurrency |
| 4 | Ch8 + Ch9 | Sub-Agents & Fork |
| 5 | Ch11 + Ch12 | Memory & Skills |
| 6 | Ch13 + Ch14 | Terminal UI & Input |
| 7 | Ch15 + Ch16 + Ch17 | MCP, Remote, Performance |
| 8 | Ch18 + Review | Epilogue & Review |

---

## CÁCH ĐỌC MỖI CHAPTER HIỆU QUẢ

### Template đọc 1 chapter:

```
1. Pre-read (10 phút):
   - Đọc title và opening paragraph
   - Scan headings và diagrams
   - Đoán: chapter này về cái gì?

2. First read (30-60 phút):
   - Đọc toàn bộ, không dừng lại quá lâu ở chỗ khó
   - Highlight những chỗ quan trọng
   - Vẽ diagram tổng quan

3. Second read (60-90 phút):
   - Đọc kỹ từng phần
   - Với mỗi code block: tái tạo trong project của mình
   - Với mỗi concept: google thêm nếu không hiểu

4. Third read (30 phút):
   - Đọc lại từ đầu
   - Giờ phải hiểu 80%+
   - Ghi ra 20% chưa hiểu để research thêm

5. Apply This (30 phút):
   - Đọc 5 patterns ở cuối
   - Với mỗi pattern: nghĩ xem "làm sao áp dụng vào project của mình?"
   - Viết ra 1-2 sentence cho mỗi pattern
```

---

## CÁC TRICK ĐẶC BIỆT CHO ENTRY-LEVEL

### 1. Khi gặp khái niệm không hiểu:

```
Bước 1: Tìm ví dụ trong cuốn sách
Bước 2: Google "what is [concept] for beginners"
Bước 3: Tìm YouTube video giải thích
Bước 4: Thử giải thích bằng lời cho người khác (hoặc cho mình)
Bước 5: Nếu vẫn không hiểu, SKIP và quay lại sau
```

### 2. Khi code block trong sách quá phức tạp:

```
Bước 1: Copy code vào editor
Bước 2: Remove comments và docstrings
Bước 3: Simplify: giữ lại interface, bỏ implementation chi tiết
Bước 4: Rewrite lại bằng cách của mình
Bước 5: Compare cách của mình với cách trong sách
```

### 3. Khi Mermaid diagram quá phức tạp:

```
Bước 1: Copy diagram text
Bước 2: Vẽ lại trên giấy từ đầu
Bước 3: Thêm arrows và notes bằng tay
Bước 4: Đi theo flow từng step một
```

### 4. Khi đọc Chapter 5 (Agent Loop):

```
Đây là chapter KHÓ NHẤT. Không cần hiểu 100%.
Mục tiêu cuối tuần:
- Hiểu tổng quan: có State, có Loop, có Compression, có Error Recovery
- Có thể trace một request qua loop
- Vẽ được diagram đơn giản
- Implement được phần nhỏ của loop
```

---

## NHỮNG CÁI BẠN SẼ HỌC ĐƯỢC

Sau khi hoàn thành roadmap này, bạn sẽ:

1. **Hiểu cách AI Agent thực sự hoạt động**
   - Không phải "magic", mà là engineering

2. **Biết cách xây một AI Agent thực tế**
   - Từ async generator loop
   - Đến tool system
   - Đến state management
   - Đến error recovery

3. **Có kiến thức nền tảng để làm việc với AI/LLM**
   - Prompt engineering
   - Tool use
   - Context management

4. **Có thể đọc và understand production code**
   - Không còn sợ codebase lớn
   - Biết cách đọc từng phần

5. **Có project để show trong CV**
   - Mini Claude Code Clone
   - MCP Server
   - Memory System

---

## LỜI KHUYÊN CUỐI CÙNG

1. **Đừng cố hiểu hết trong lần đầu.** Đọc nhiều lần, mỗi lần hiểu thêm một chút.

2. **Luôn code thực tế.** Mỗi concept bạn đọc được, thử implement nó đơn giản.

3. **Tập trung vào PATTERNS, không phải implementations.** Cuốn sách dạy bạn CÁCH NGHĨ, không phải CÁCH COPY.

4. **Vẽ diagram.** Mỗi chapter, vẽ diagram bằng tay ít nhất 3 lần.

5. **Dạy người khác.** Cách tốt nhất để học là explain cho người khác.

6. **Đừng so sánh mình với senior.** Entry-level là entry-level. Senior cũng từng ở đây.

---

**Chúc bạn học tốt! 🚀**