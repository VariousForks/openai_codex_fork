# How Tooling is Used in Codex CLI

This document explains how Codex CLI integrates with OpenAI to execute external commands (tools) during an interactive coding session. It covers:
- Legacy chat-based function calls and their quadratic token cost
- The current incremental `/responses` API with function-calling
- Reasoning models (e.g. `o3`, `o4-mini`) and built-in summarization
- Core code flow, key classes/functions
- A Mermaid diagram illustrating the agent loop

---

## 1. Legacy Chat-based Function Calls

In earlier implementations, every new tool call required re-sending the entire conversation history via the Chat Completions API. Each request looked like:
```js
// (pseudo-code)
openai.chat.completions.create({
  model: 'gpt-4',
  messages: fullConversationHistory,
  functions: [ /* tool definitions */ ],
});
```
Because the client had to include all previous messages each turn, token usage grew linearly per turn, and total cost was _quadratic_ in the number of turns.

---

## 2. Current Incremental `/responses` API with Function Calling

Codex CLI now leverages the new `/responses` endpoint, which maintains server-side conversation state and accepts only the _new_ inputs each turn. Context is preserved via a `previous_response_id` rather than re-sending all history.

### Key request parameters (in `AgentLoop.run`)
```ts
// File: codex-cli/src/utils/agent/agent-loop.ts
const stream = await this.oai.responses.create({
  model: this.model,
  instructions: mergedInstructions,
  previous_response_id: lastResponseId || undefined,
  input: turnInput,                // only new items (user input, tool outputs)
  stream: true,
  parallel_tool_calls: false,
  reasoning,                       // e.g. { effort: 'high', summary: 'auto' }
  tools: [                         // function definitions
    {
      type: 'function',
      name: 'shell',
      description: 'Runs a shell command, and returns its output.',
      parameters: { /* JSON schema for command, workdir, timeout */ },
    },
  ],
});
```

The server returns a streaming response of events. When the model emits a `function_call` event, the client handles it without restarting the entire conversation:

```ts
// Upon receiving a function_call event:
if (item.type === 'function_call') {
  this.pendingAborts.add(item.call_id || item.id);
} else {
  stageItem(item);
}
```

After the reasoning/pass completes, any `function_call` items are processed in
`handleFunctionCall`, which invokes the tool and returns a `function_call_output` item to be sent back in the _next_ increment. No full-history payload is needed.

---

## 3. Function Call Handling

```ts
// File: codex-cli/src/utils/agent/agent-loop.ts
private async handleFunctionCall(
  item: ResponseFunctionToolCall
): Promise<Array<ResponseInputItem>> {
  // parse name & args
  const args = parseToolCallArguments(rawArguments);
  // default output
  const outputItem: ResponseInputItem.FunctionCallOutput = {
    type: 'function_call_output',
    call_id: callId,
    output: 'no function found',
  };
  
  // for shell or container.exec calls:
  if (name === 'shell' || name === 'container.exec') {
    const { outputText, metadata, additionalItems } =
      await handleExecCommand(
        args,
        this.config,
        this.approvalPolicy,
        this.additionalWritableRoots,
        this.getCommandConfirmation,
        this.execAbortController?.signal,
      );
    outputItem.output = JSON.stringify({ output: outputText, metadata });
    return [outputItem, ...(additionalItems || [])];
  }
  return [outputItem];
}
```

The helper `handleExecCommand` is defined in:
```ts
// File: codex-cli/src/utils/agent/handle-exec-command.ts
export async function handleExecCommand(
  args: ExecInput,
  config: AppConfig,
  policy: ApprovalPolicy,
  additionalWritableRoots: string[],
  getCommandConfirmation: (...)
): Promise<HandleExecCommandResult> { /* runs shell, sandboxing, approval */ }
```

---

## 4. Reasoning Models & Summarization

For models whose names start with `o` (e.g. `o3`, `o4-mini`), Codex CLI passes an optional `reasoning` parameter:
```ts
let reasoning: Reasoning = { effort: 'high' };
if (this.model === 'o3' || this.model === 'o4-mini') {
  reasoning.summary = 'auto';
}
```
This instructs the server to automatically summarize earlier parts of the conversation, further reducing token usage and preventing unbounded context growth.

---

## 5. Agent Loop Diagram

```mermaid
flowchart TD
  subgraph User Turn
    U[User Input] --> RL[AgentLoop.run()]
  end
  RL --> Req[OpenAI /responses.create]
  Req -->|stream| Evt[Streaming Events]
  Evt -->|message| Display[Display to User]
  Evt -->|function_call| FC[handleFunctionCall]
  FC --> Exec[execute shell/container]
  Exec --> RespOut[function_call_output]
  RespOut --> RL[AgentLoop.run()]  
  Display --> End[End Turn]
```

---

## 6. Further Reading

- **AgentLoop**: `codex-cli/src/utils/agent/agent-loop.ts` (methods: `run()`, `handleFunctionCall()`)
- **OpenAI Client Setup**: same file (`new OpenAI().responses.create`)
- **Exec Handling**: `codex-cli/src/utils/agent/handle-exec-command.ts`
- **Parsers**: `codex-cli/src/utils/parsers.ts` (JSON argument parsing)

This flow ensures linear token growth, integration of server-side summarization for reasoning models, and seamless tool invocation without re-sending the full chat history each turn.