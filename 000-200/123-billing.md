Here’s the quick way to think about it:

* The **Vercel Chatbot** template on GitHub is a Next.js app built on the **Vercel AI SDK**; you hook your token metering right where the SDK streams or finishes a response. ([GitHub][1])
* The AI SDK already **exposes token usage** for you; you just need to read it and store it. Use `onFinish` / `onStepFinish` (and optionally `onAbort`) or the `totalUsage` helper. ([ai-sdk.dev][2])

Below are the three practical ways to measure tokens with this stack (plus copy-pasteable snippets).

---

# 1) Server-side (recommended): record usage from the AI SDK

### A. In your threads route (e.g. `app/api/threads/route.ts`)

```ts
import { openai } from '@ai-sdk/openai';
import {
  streamText,
  convertToModelMessages,
  type UIMessage,
} from 'ai';
import { db } from '@/lib/db'; // your persistence
import { usageLog } from '@/lib/schema'; // your table

export async function POST(req: Request) {
  const { messages, tenantId }: { messages: UIMessage[]; tenantId: string } =
    await req.json();

  const result = streamText({
    model: openai('gpt-4o'),           // any provider works here
    messages: convertToModelMessages(messages),
    // Called if the stream is aborted (user closed tab, clicked stop, etc.)
    onAbort: async ({ steps }) => {
      // Optional: sum per-step usage to capture partial consumption
      const totals = steps.reduce(
        (acc, s) => {
          const u = s.usage ?? { promptTokens: 0, completionTokens: 0, totalTokens: 0 };
          acc.promptTokens += u.promptTokens ?? u.inputTokens ?? 0;
          acc.completionTokens += u.completionTokens ?? u.outputTokens ?? 0;
          acc.totalTokens += u.totalTokens ?? 0;
          return acc;
        },
        { promptTokens: 0, completionTokens: 0, totalTokens: 0 },
      );
      if (totals.totalTokens > 0) {
        await db.insert(usageLog).values({
          tenantId,
          model: 'openai/gpt-4o',
          promptTokens: totals.promptTokens,
          completionTokens: totals.completionTokens,
          totalTokens: totals.totalTokens,
          wasAborted: true,
        });
      }
    },
  });

  // Stream the response to the client and record *final* usage on completion:
  return result.toUIMessageStreamResponse({
    async onFinish({ totalUsage, isAborted }) {
      if (isAborted) return; // already handled in onAbort
      const { promptTokens, completionTokens, totalTokens } = totalUsage;
      await db.insert(usageLog).values({
        tenantId,
        model: 'openai/gpt-4o',
        promptTokens,
        completionTokens,
        totalTokens,
        wasAborted: false,
      });
    },
  });
}
```

* `onFinish` receives **`totalUsage`** (sum across multi-step/tool-calling runs) so your numbers are correct even when the model calls tools. In AI SDK v5, `usage` is per last step; use **`totalUsage`** for billing. ([ai-sdk.dev][3])
* If the user **aborts** a stream, `onFinish` may not fire; use **`onAbort`** and sum `steps[i].usage` to capture partial usage. ([ai-sdk.dev][3])

> Docs that show this pattern and the shape of the usage object (prompt/completion/total): ([ai-sdk.dev][2])

---

# 2) Pre-flight estimates (before you send the request)

Sometimes you want to **estimate** tokens to enforce limits or quote costs before sending:

* **OpenAI-family models:** use a local tokenizer (e.g., `tiktoken` or `gpt-tokenizer`) to estimate context tokens.

  ```ts
  import { encode } from 'gpt-tokenizer';
  const estimate = encode(text).length; // rough estimate for GPT-style models
  ```

  (Estimates can differ slightly from billed usage.) ([OpenAI Cookbook][4])

* **Anthropic (Claude):** they provide an official **Count Tokens API**—ask for token count without generating. ([Anthropic][5])

* **Google Gemini (Vertex/AI Studio):** use **CountTokens** to precompute input tokens. ([Google Cloud][6])

Pre-flight is great for **guardrails**, but for **billing** you should still record the **server-reported usage** from §1.

---

# 3) Zero-code metering via **Vercel AI Gateway** (optional but nice)

If you route your AI calls through **Vercel AI Gateway**, you’ll automatically get **Input/Output Token Counts** and spend charts in the Vercel dashboard. Two easy ways:

* In AI SDK v5, you can specify models as plain strings (e.g., `'openai/gpt-4o'`) and set `AI_GATEWAY_API_KEY`. The SDK speaks to the Gateway provider and you get token metrics in **Observability → AI Gateway**. ([Vercel][7])
* Or create a Gateway provider with `@ai-sdk/gateway` and use it explicitly. Default base URL: `https://ai-gateway.vercel.sh/v1/ai`. ([ai-sdk.dev][8])

This is a solid complement to your own DB logs (nice for dashboards and cross-project rollups). ([Vercel][9])

---

## Where this plugs into the **vercel/ai-threadsbot** repo

In that template, the AI call happens in the server layer that streams threads responses (a Next.js route or server action that calls `streamText`). Add the `onFinish` / `onAbort` logic shown above and persist usage next to your messages table. ([GitHub][1])

---

## TL;DR billing flow you can ship

1. **On every response**: read `totalUsage` in `onFinish` and insert `{tenantId, model, promptTokens, completionTokens, totalTokens}` into a `usage_ledger`. Handle `onAbort` for partial usage. ([ai-sdk.dev][3])
2. **Convert tokens → credits** using your price map per model (e.g., tokens / 1000 × price) and **debit** the customer’s prepaid balance.
3. (Optional) **Pre-flight** check with a tokenizer / provider count-tokens endpoint to reject/trim prompts that would exceed the customer’s balance. ([OpenAI Cookbook][4], [Anthropic][5], [Google Cloud][6])
4. (Optional) Turn on **Vercel AI Gateway** for a second source of truth and dashboards. ([Vercel][9])

If you want, I can sketch the minimal Drizzle table + a helper `recordUsage(tenantId, totalUsage, model)` wired into your route.

[1]: https://github.com/vercel/ai-threadsbot "GitHub - vercel/ai-threadsbot: A full-featured, hackable Next.js AI threadsbot built by Vercel"
[2]: https://ai-sdk.dev/cookbook/rsc/stream-ui-record-token-usage "React Server Components: Record Token Usage after Streaming User Interfaces"
[3]: https://ai-sdk.dev/docs/advanced/stopping-streams?utm_source=threadsgpt.com "Advanced: Stopping Streams"
[4]: https://cookbook.openai.com/examples/how_to_count_tokens_with_tiktoken?utm_source=threadsgpt.com "How to count tokens with Tiktoken"
[5]: https://docs.anthropic.com/en/api/messages-count-tokens?utm_source=threadsgpt.com "Count Message tokens"
[6]: https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/list-token?utm_source=threadsgpt.com "List and count tokens | Generative AI on Vertex AI"
[7]: https://vercel.com/docs/ai-gateway/authentication?utm_source=threadsgpt.com "Authentication"
[8]: https://ai-sdk.dev/providers/ai-sdk-providers/ai-gateway?utm_source=threadsgpt.com "AI Gateway Provider"
[9]: https://vercel.com/docs/ai-gateway/observability?utm_source=threadsgpt.com "Observability"
