# Durable Actors

> _an open-source alternative to Cloudflare Durable Objects... without the vendor lock-in, memory limits, and terrible observability_

Durable Actors helps you **build real-time applications** like chat systems (e.g. ChatGPT, Codex), collaboration tools (e.g. Notion), and agent swarms (e.g. Devin).

It provides _stateless serverless functions_, a foundational building block that abstracts away persistence, coordination, and infrastructure challenges in distributed systems.

## How it works

1. Define an _actor_, a class with _durable state_ (i.e. data survives interruptions, errors, and restarts) and _serialized execution_ (i.e. concurrent callers can update it safely).
2. Generate type-safe clients automatically with the Durable Actors SDK. For now, it supports Python and TypeScript (with WebSocket under the hood).
3. Develop locally with one command and later self-host the Durable Actors runtime for production.

For example:

- **If you were building ChatGPT...** a chat actor can store conversations that survive LLM flakiness and server crashes (durable state)
- **If you were building Notion...**  a document actor can coordinate concurrent edits from several people and agents (serialized execution)

## Quickstart: Multiplayer AI Chat

<div align="left">
  <a href="https://github.com/TerseAI/durable-actors/blob/main/.github/assets/team-agent.gif">
    <picture>
      <source media="(prefers-reduced-motion: reduce)" srcset=".github/assets/team-agent.png">
      <img alt="Teammates share one TeamAgent chat across regions; prompts queue, replies stream to everyone, and conversation state is durably persisted." src=".github/assets/team-agent.gif" width="1000">
    </picture>
  </a>
</div>

### 1. Create your project

Install Node.js 22.19+, pnpm, and Bun 1.3.9+.

```sh
npx durable-actors init my-actors
cd my-actors
pnpm install
# Or with npm:
npm install
npx durable-actors dev # Run the server locally on your machine
```

### 2. Define an _actor_

Define and export actors in your actor project’s `src/actors.ts`, the default entrypoint loaded by `durable-actors dev`. The runtime loads actors on demand and persists fields marked `@Persisted`. For example, a chat history actor:

```ts
import { openai } from "@ai-sdk/openai"
import { streamText } from "ai"
import { Actor, Persisted, Reentrant, type ActorSocket } from "durable-actors"

type Member = { name: string }
type Message = { role: "user" | "assistant"; content: string }
type Chat = { messages: Message[]; busy: boolean }

export class ChatHistory extends Actor<Member, string, Chat> {
    @Persisted messages: Message[] = []

    async onConnect(socket: ActorSocket<Member, Chat>) {
        socket.send({ messages: this.messages, busy: false })
    }

    @Reentrant
    async onMessage(socket: ActorSocket<Member, Chat>, text: string) {
        const messages: Message[] = [...this.messages, { role: "user", content: `${socket.metadata.name}: ${text}` }]
        this.broadcast({ messages, busy: true })

        const reply: Message = { role: "assistant", content: "" }
        const result = streamText({ model: openai("gpt-5-mini"), messages })
        for await (const chunk of result.textStream) {
            reply.content += chunk
            this.broadcast({ messages: [...messages, reply], busy: true })
        }

        this.messages = [...messages, reply]
        this.broadcast({ messages: this.messages, busy: false })
    }
}
```


### 3. Connect your backend

We make it super easy to integrate the actors into your existing tech stack. Just generate the client and you get a fully type safe contract to interact with.

```sh
npx durable-actors generate
```

Now you may call your actor and access the state.

```ts
import express from "express"

import { actors } from "../generated/index.js"

export const app = express()

app.post("/api/chat/:room/socket", async (req, res) => {
    const grant = await actors.ChatHistory.prepareWebsocket({
        actorId: req.params.room,
        metadata: { name: String(req.query.name ?? "Guest") }
    })
    res.set("Cache-Control", "no-store").json(grant)
})
```

### 4. Connect the frontend

```tsx
import { useEffect, useRef, useState } from "react"
import { createRoot } from "react-dom/client"

import type { actors } from "../generated/index.js"

const params = new URLSearchParams(location.search)
const room = params.get("chat") ?? "lobby"
const name = params.get("name") ?? "Guest"

function Chat() {
    const socket = useRef<WebSocket>(null)
    const [chat, setChat] = useState<actors.ChatHistory.Outgoing>({ messages: [], busy: true })

    useEffect(() => {
        let active = true
        async function connect() {
            const response = await fetch(`/api/chat/${encodeURIComponent(room)}/socket?name=${encodeURIComponent(name)}`, { method: "POST" })
            const { websocketUrl } = await response.json()
            if (!active) return
            socket.current = new WebSocket(websocketUrl)
            socket.current.onmessage = event => setChat(JSON.parse(event.data))
        }
        void connect()
        return () => {
            active = false
            socket.current?.close()
        }
    }, [])

    function send(form: FormData) {
        const text = String(form.get("message")).trim()
        if (!text || socket.current?.readyState !== WebSocket.OPEN) return
        socket.current.send(JSON.stringify(text))
        setChat(chat => ({ ...chat, busy: true }))
    }

    return (
        <main>
            <h1>AI chat · {room}</h1>
            <div role="log" aria-label="Messages">
                {chat.messages.map((message, index) => (
                    <article key={index}>
                        <strong>{message.role}</strong>
                        <p>{message.content}</p>
                    </article>
                ))}
            </div>
            <form action={send}>
                <input name="message" aria-label="Message" required disabled={chat.busy} />
                <button disabled={chat.busy}>Send</button>
            </form>
        </main>
    )
}

createRoot(document.getElementById("root")!).render(<Chat />)
```

### 5. Monitor and debug

Durable Agents provides built-in observability features:

<img width="1599" height="676" alt="o11y-screenshot" src="https://github.com/user-attachments/assets/ac390257-8534-4911-830c-0e9155b36831" />

## Examples

[![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white)](sdk/README.md) [![Python](https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white)](sdk-python/README.md)

For complete sample applications, see [AI Chat](examples/ai-chat), [Collaborative Documents](examples/documents), and [Chatroom](examples/chat).

## Community

Bug reports, feature requests, documentation fixes, and code contributions are welcome. See the [contributing guide](CONTRIBUTING.md) for repository setup and checks, and follow our [code of conduct](CODE_OF_CONDUCT.md).

Use [GitHub Issues](https://github.com/TerseAI/durable-actors/issues) for bugs, ideas, and questions. Report vulnerabilities privately using our [security policy](SECURITY.md).

Follow development and release notes on [GitHub Releases](https://github.com/TerseAI/durable-actors/releases).

## License

[MIT](LICENSE.md) © 2026 Terse
