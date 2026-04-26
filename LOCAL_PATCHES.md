# Local Patches

Tracks local modifications to the upstream `deliberate-lab` source. These are
not (yet) upstreamed — keep this file when pulling from `main` and reapply
patches as needed.

---

## 2026-04-26: Fix multi-agent chat dropping most agent responses

**File:** `functions/src/chat/chat.agent.ts`

### Symptom
With ≥2 agent participants of the same `UserType` (e.g., `participant`)
sharing a chat stage, only one agent's response per trigger message was
persisting to Firestore. Cloud Functions logs showed many `"Conversation has
moved on"` entries — generated LLM responses were being silently dropped.

### Root cause
Two gates in `sendAgentGroupChatMessage` / `sendAgentPrivateChatMessage`:

1. **"Moved on" check** (strict): dropped any response if the last chat
   message was no longer the agent's `triggerChatId`. With multiple agents
   running parallel LLM calls, the first to finish posts; everyone else
   wakes up to a moved-on chat and is dropped.
2. **Type-keyed slot lock**: trigger-log doc keyed on
   `${triggerChatId}-${chatMessage.type}`. All agent participants share
   `type === "participant"`, so the first one to claim the slot blocks
   every other agent of the same type from posting to that trigger.

### Patch

- Introduced `MAX_MOVED_ON_MESSAGE_GAP = 5` constant (top of file). Soften
  the moved-on check from "must still be the latest message" to "must be
  within N messages of the end." Set to `0` to restore strict behavior.
- Re-keyed the slot-lock from `chatMessage.type` to `chatMessage.senderId`,
  giving every agent its own per-trigger slot instead of one slot shared
  by all agents of the same type.
- Improved drop-log lines to include sender id and message-gap count.

### Trade-offs

- ~3× more LLM-generated messages get persisted (previously ~2/3 were
  dropped in 3-agent setups). Expect higher Firestore writes and LLM cost.
- Agents may now reply to triggers that are 1–5 messages stale. Their
  prompts still see the up-to-date chat history at LLM time; only
  persistence is affected.
- `MAX_MOVED_ON_MESSAGE_GAP = 5` is a hardcoded guess. Tune per experiment;
  consider promoting it to a configurable field on `AgentChatSettings` if
  experimenters want per-stage control.

### Reapply / verify after upstream pull

```bash
cd functions
npx tsc --noEmit src/chat/chat.agent.ts   # should be clean
grep -n "MAX_MOVED_ON_MESSAGE_GAP\|chatMessage.senderId" src/chat/chat.agent.ts
# expect: 1 declaration + 2 uses of the constant; 2 uses of senderId in
# trigger-log keys (group + private).
```

### Related code references

- Trigger fan-out (parallel `Promise.all`): `functions/src/triggers/chat.triggers.ts:81-110`
- Group chat sender (patched): `functions/src/chat/chat.agent.ts` — `sendAgentGroupChatMessage`
- Private chat sender (patched): `functions/src/chat/chat.agent.ts` — `sendAgentPrivateChatMessage`
- Typing-delay helper (unchanged): `utils/src/shared.ts` — `awaitTypingDelay`, `getTypingDelayInMilliseconds`
