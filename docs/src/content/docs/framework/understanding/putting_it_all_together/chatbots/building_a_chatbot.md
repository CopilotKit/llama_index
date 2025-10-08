---
title: How to Build a Chatbot
---

This guide shows how to build a production-ready chatbot that combines LlamaIndex agents with CopilotKit for a real application experience: chat UI, runtime bridge, shared state, frontend actions, human-in-the-loop, and optional multi-agent routing. We’ll start fresh and focus on CopilotKit-first integration.

### Prerequisites

- Node.js 20+
- Python 3.10+
- An OpenAI API key in your environment (`OPENAI_API_KEY`)

### Architecture Overview

You’ll run a small Python service that exposes your LlamaIndex agent via the AG-UI protocol. A Next.js route will host the Copilot Runtime that bridges your frontend to the agent. CopilotKit’s React components provide the chat UI and advanced interaction patterns.

Overview:

1. Python: FastAPI + LlamaIndex AG-UI router (your agent service)
2. Next.js: Copilot Runtime endpoint connects to the agent
3. React: CopilotKit provider + chat UI
4. Advanced: Frontend actions, shared state, human-in-the-loop, multi-agent flows

---

## 1) Create your LlamaIndex agent service (Python)

Create `agent.py`:

```python
import os
from fastapi import FastAPI
from llama_index.llms.openai import OpenAI
from llama_index.protocols.ag_ui.router import get_ag_ui_workflow_router

# Configure your LLM
llm = OpenAI(
    model="gpt-4.1",  # or "gpt-4o"
    api_key=os.getenv("OPENAI_API_KEY"),
)

# Optional: declare frontend-executed tools (names/signatures are exposed to the LLM)
def say_hello(name: str) -> str:
    """Say hello to the user (executed on the frontend via CopilotKit)."""
    return f"Hello, {name}!"

def write_essay(draft: str) -> str:
    """Prepare an essay draft and ask the user to approve it (HITL)."""
    return "Draft prepared. Please review."

app = FastAPI(title="LlamaIndex Agent", version="1.0.0")

agentic_chat_router = get_ag_ui_workflow_router(
    llm=llm,
    system_prompt=(
        "You are a helpful AI assistant. Use tools wisely and ask for human "
        "confirmation when appropriate."
    ),
    # Shared state is automatically injected into LLM context
    initial_state={
        "language": "english",
        "tone": "friendly",
    },
    # Frontend actions (will execute in the browser via CopilotKit)
    frontend_tools=[say_hello, write_essay],
)

app.include_router(agentic_chat_router)

@app.get("/health")
async def health_check() -> dict:
    return {"status": "healthy", "agent": "llamaindex"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

Run it (in a dedicated terminal):

```bash
export OPENAI_API_KEY="your_openai_api_key"
python agent.py
```

---

## 2) Create a Copilot Runtime endpoint (Next.js)

Add `app/api/copilotkit/route.ts` to your Next.js app:

```ts
import {
  CopilotRuntime,
  ExperimentalEmptyAdapter,
  copilotRuntimeNextJSAppRouterEndpoint,
} from "@copilotkit/runtime";
import { LlamaIndexAgent } from "@ag-ui/llamaindex";
import { NextRequest } from "next/server";

// Router for single-agent setups. For multi-agent, use a real adapter.
const serviceAdapter = new ExperimentalEmptyAdapter();

const runtime = new CopilotRuntime({
  agents: {
    llamaindex_agent: new LlamaIndexAgent({ url: "http://localhost:8000/run" }),
  },
});

export const POST = async (req: NextRequest) => {
  const { handleRequest } = copilotRuntimeNextJSAppRouterEndpoint({
    runtime,
    serviceAdapter,
    endpoint: "/api/copilotkit",
  });
  return handleRequest(req);
};
```

---

## 3) Wrap your app with the CopilotKit provider (React)

In `app/layout.tsx`:

```tsx
import { CopilotKit } from "@copilotkit/react-core";
import "@copilotkit/react-ui/styles.css";
import "./globals.css";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <CopilotKit runtimeUrl="/api/copilotkit" agent="llamaindex_agent">
          {children}
        </CopilotKit>
      </body>
    </html>
  );
}
```

---

## 4) Add a chat UI

In `app/page.tsx`:

```tsx
import { CopilotChat } from "@copilotkit/react-ui";

export default function Page() {
  return (
    <main className="min-h-screen p-8 bg-indigo-50">
      <CopilotChat
        labels={{ initial: ["Howdy! I’m connected to your LlamaIndex agent 🦙"] }}
        className="w-[600px] h-[80vh] mx-auto border rounded-xl p-4"
      />
    </main>
  );
}
```

---

## 5) Frontend Actions (tools executed in the browser)

Register actions with `useCopilotAction`. When the agent decides to call them, CopilotKit executes your React handler.

```tsx
import { useCopilotAction } from "@copilotkit/react-core";

export function FrontendActions() {
  useCopilotAction({
    name: "say_hello",
    description: "Say hello to the user",
    parameters: [
      { name: "name", type: "string", description: "User’s name", required: true },
    ],
    handler: async ({ name }) => {
      alert(`Hello, ${name}!`);
    },
  });
  return null;
}
```

Include `<FrontendActions />` anywhere under the `CopilotKit` provider so the action is registered.

On the Python side, ensure the function name/signature exists in `frontend_tools=[say_hello, ...]` so the LLM knows it can call it.

---

## 6) Shared state between your app and the agent

State defined in `initial_state` is available to the LLM. You can read and update it from the frontend using `useCoAgent`.

Agent state defined earlier:

```python
agentic_chat_router = get_ag_ui_workflow_router(
    llm=llm,
    initial_state={"language": "english", "tone": "friendly"},
)
```

Frontend usage:

```tsx
import { useCoAgent } from "@copilotkit/react-core";

type AgentState = {
  language: "english" | "spanish";
  tone: "friendly" | "formal";
};

export function StateControls() {
  const { state, setState, run } = useCoAgent<AgentState>({ name: "llamaindex_agent" });

  const toggleLanguage = () => {
    const next = state.language === "english" ? "spanish" : "english";
    setState({ language: next });
    // Optionally re-run with a hint about what's changed
    run(({ currentState }) => ({
      id: "state-change-language",
      role: "developer",
      content: `language updated to ${currentState.language}`,
    }));
  };

  return (
    <div className="flex gap-2">
      <span>Language: {state.language}</span>
      <button onClick={toggleLanguage}>Toggle Language</button>
    </div>
  );
}
```

---

## 7) Human-in-the-Loop (HITL)

Let the agent hand control to the UI to collect user approval or input before continuing.

```tsx
import { useCopilotAction } from "@copilotkit/react-core";
import { Markdown } from "@copilotkit/react-ui";

export function HitlWidgets() {
  useCopilotAction({
    name: "write_essay",
    description: "Writes an essay and asks for approval.",
    parameters: [
      { name: "draft", type: "string", description: "Essay draft", required: true },
    ],
    followUp: false,
    renderAndWaitForResponse: ({ args, respond, status }) => (
      <div>
        <Markdown content={args.draft || "Preparing your draft..."} />
        <div className={`flex gap-4 pt-4 ${status !== "executing" ? "hidden" : ""}`}>
          <button onClick={() => respond?.("CANCEL")} className="border p-2 rounded-xl w-full">
            Try Again
          </button>
          <button onClick={() => respond?.("SEND")} className="bg-blue-500 text-white p-2 rounded-xl w-full">
            Approve Draft
          </button>
        </div>
      </div>
    ),
  });
  return null;
}
```

On the agent side, the `write_essay` tool signature in `frontend_tools` is sufficient—AG-UI will return control to the UI when the LLM chooses this tool.

---

## 8) Multi-agent flows (optional)

CopilotKit supports two modes:

- Router Mode (default): don’t pass an `agent` prop to `CopilotKit`. The runtime routes between agents/tools.
- Agent Lock Mode: pass `agent="llamaindex_agent"` to lock to a single agent workflow.

Router Mode example (`app/layout.tsx`):

```tsx
<CopilotKit runtimeUrl="/api/copilotkit">
  {children}
</CopilotKit>
```

Agent Lock Mode example:

```tsx
<CopilotKit runtimeUrl="/api/copilotkit" agent="llamaindex_agent">
  {children}
</CopilotKit>
```

Use Router Mode for chat-first experiences with multiple capabilities; use Agent Lock when you want precise, single-workflow control.

---

## 9) Run everything

Open two terminals:

Terminal 1 (agent):

```bash
export OPENAI_API_KEY="your_openai_api_key"
python agent.py
```

Terminal 2 (Next.js):

```bash
npm run dev
```

Visit `http://localhost:3000` and start chatting.

---

## 10) Troubleshooting

- Agent health: `curl http://localhost:8000/health`
- CORS/host issues: try `127.0.0.1` instead of `localhost`
- Runtime URL: ensure `runtimeUrl` matches your Next.js route
- Tool not firing: confirm the tool name in `useCopilotAction` matches the Python function name in `frontend_tools`
- Shared state not syncing: ensure `initial_state` is set in the router and you’re using `useCoAgent` under the same provider
