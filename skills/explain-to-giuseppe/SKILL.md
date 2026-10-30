---
name: explain-to-giuseppe
description: >
  Explain concepts, code, protocols or product behavior the way Giuseppe learns:
  as a short step-by-step story told from one actor's point of view, with causes,
  plain words first and the technical term in parentheses. Use whenever Giuseppe
  asks to explain, unpack, walk through or help understand something, or says an
  explanation "didn't click" or "the penny hasn't dropped".
---

# Explain to Giuseppe

Giuseppe understands by following a story, not by reading definitions. In the session this
came from, a table of protocol events between five parties did not land. Giuseppe's own
retelling, as a numbered story told from one actor's point of view, did. Explain in that second
way. (In that session the actor was an AI agent, but it can be any party: a user, a server, a
function.)

## Shape of every answer (most important)

Giuseppe stops reading or skips ahead when text looks long. Anything past that point is lost.
So make the shape do the work.

- **First sentence = the main point.** Don't build up to it.
- **Main concepts first.** No edge cases, caveats, history or options unless asked.
- **Details only on request.** Giuseppe likes details and will ask when ready. Offer them in one
  line ("Want the details?"). Don't add them unasked.
- **Short sentences.** One idea each.
- **Short paragraphs.** Two or three lines at most. No walls of text.
- **Prefer a few numbered lines to long prose.**
- **Anything optional goes last**, clearly separated, so skipping it loses nothing.
- **End with one question or next step**, not several.

If an answer looks long, cut it. Keep the one idea that matters.

## How to explain

1. **One protagonist, in order.** Tell it from the point of view of one actor (the agent, the
   user, the server) as numbered steps. For each step: what this actor does, and why. Stay with
   that actor. Don't hop between several.
2. **Causes, not just events.** Join steps with "because" and "so": "the API finds no token, so
   it answers 'not authorized'."
3. **Start from something concrete** that Giuseppe has or built: a route in the app, a config
   value, a screen already seen. Not from a standard or a definition.
4. **Plain words first, the technical term in parentheses.** Write the plain version, then put
   the protocol term right after it. Example: "send (redirect, HTTP 302) the human to our own
   login page (the Authentication URL)". Never drop the technical term, and never lead with it.
5. **Say where each piece of information comes from.** For every URL, value or fact: who has
   it, and how did the other party learn it (configured in advance, found by discovery, told in
   a response)? Giuseppe's questions are usually of this kind: "how does X know Y?"
6. **Fix the names and keep them.** Use one name per actor for the whole explanation (for
   example "Support Agent" and "AI agent"), and never one word for two things. Giuseppe notices
   naming inconsistencies, so fix them.
7. **One step at a time, and short.** After a step or a small group of steps, stop and let
   Giuseppe retell it. Don't give everything at once.
8. **When Giuseppe retells it, check it against the facts.** Say what is right. Correct only
   what is wrong, in the same story form, and add missing details as small additions. Don't
   re-explain from scratch.
9. **Analogies are welcome** (a house, a hotel front desk, a keycard), but map the analogy back
   to the real parts, and don't let it replace them.
10. **Tables and option lists come after the story, never instead of it.** A table of events
    across many parties is a reference. Use it only once the story has landed.
11. **Say what is unknown.** If something is an inference and not documented, say so in one
    line.

## What does not work

- A table of protocol events between many parties at once.
- Starting with definitions or standards (RFC numbers, protocol names) before the story needs
  them.
- Protocol words in the main sentence (302, Set-Cookie, "front channel").
- Offering several options when one recommendation would do.
- A long first paragraph. Giuseppe reads a few lines, then skips to the next paragraph.
- Side notes, caveats and extra detail in the same answer as the main point.

## Example

Didn't land:

> Browser → Acme Identity: GET /oauth2/authorize. Acme Identity → Browser: 302 to the Authentication URL with
> a nonce and a cookie. Browser → app: GET /login...

Landed:

1. The AI agent tries to read charges from our API. It has no token yet.
2. Our API's check (`authorize()`) fails because nothing was sent, so the API answers "not
   authorized" (HTTP 401).
3. The AI agent understands it needs to get authorized, so it looks in its own config and finds
   the authorization server (the auth domain).
4. It builds the sign-in address (the authorize URL) from that, and opens a browser for the
   human (the Support Agent).
5. That address belongs to Acme Identity, the authorization server. Its first step is to send (redirect, HTTP 302)
   the human to our own login page (the Authentication URL).
