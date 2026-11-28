---
name: chat-handover
description: Use when a conversation has become very long, after finishing a significant feature or fix, when the user asks for a handover, or when Claude is unsure about something settled earlier in the chat. Writes a versioned handover file that lets a brand-new chat or project chat continue the work without re-reading this one: summary, decisions, state, what was verified, files to bring, environment to rebuild, chat name and a short first message.
---

# Chat Handover

Long chats get heavier with every message because the whole conversation is re-read each time. This skill moves the work to a fresh chat without losing anything. The test of a good handover: **a fresh Claude with only the handover and the attached files could carry on without asking what happened.**

## When to use

- A significant feature or fix has just been completed
- The conversation has become very long, with many rounds of revisions
- Claude is unsure about something settled earlier, or notices it is repeating itself or contradicting earlier decisions
- The user asks for a handover

If the chat is still productive and the user would rather continue, say what the trade-off is and let them choose. Do not push a restart.

## What to do

1. **Suggest a new chat and say why** in one or two sentences.

2. **Check what is true before writing.** Look at the actual files and the last test or build results rather than relying on memory. If something cannot be confirmed, write it down as unverified.

3. **Write the handover as a downloadable, versioned file** (for example `project-chat-handover-v1.md`), not only as chat text. Chat text is lost when the chat ends. If the project has its own handover or README in the repo, point to it instead of copying it: the chat handover is the short "start here", the repo handover is the detail. Use the template below.

4. **Prepare the "files to bring" list.** Go back through the whole conversation and cover BOTH:
   - Files Claude created: the latest version of every HTML, code, zip or document
   - Files the user uploaded: specs, PDFs, floor plans, CSV exports, screenshots, reference documents and any other attachments. Uploads do not carry over to a new chat.

   For each file give:
   - Name and current version number
   - Size and a short checksum (first 12 characters of sha256), so the new chat can confirm it was given the right version
   - Whether the new chat still needs it (essential / optional / no)
   - Where it should live: **project knowledge** for stable files (specs, the handover), **attached to the first message** for files that change (the code zip, the chat handover)
   - Whether it is already in project knowledge or must be re-uploaded

   Also:
   - Flag any file whose latest version exists only in this chat, and make sure it has actually been delivered as a downloadable file now. If it cannot be delivered, output it in full.
   - List superseded files by name and tell the user to remove them from project knowledge so they do not conflict.
   - Say whether anything pasted into the chat (not uploaded) is also needed, and where a copy is.

5. **Use consistent version numbers.**
   - Name files with the project and a version number, e.g. `telco-earth-v12.html`
   - Increase the number every time a new build is handed over; never reuse a number
   - If earlier deliveries had no version, give them a number now and state that the unversioned copies are the same content, so only the numbered names are used from now on
   - State the number the next build must use

6. **Suggest a name for the next chat:** the project name plus the next part number, e.g. "Telco Earth part 3".

7. **Finish with a short first message** for the new chat. It should name the chat, list the files to attach, point to the handover file instead of repeating it, ask the new chat to confirm the starting state before changing anything, state the next version number, and leave a gap for today's goal.

## Handover template

Use these headings, in this order. Keep each short; link to the repo handover for detail.

1. **Why a new chat** (one short paragraph)
2. **What we have built** (one paragraph, then a short list of the main parts)
3. **Current state** (what works now; the last test or build result, with numbers)
4. **How it is put together** (only what is needed to change anything: where things live, the main entry points)
5. **Decisions made, and why** (a numbered list; label them "do not undo without a reason")
6. **Known issues and unverified things** (two clear lists: known bugs, and what was never tested, such as real devices or real users)
7. **Next steps** (suggested; say none are started if none are)
8. **Working agreements** (how the user likes to work, as seen in this chat: e.g. decide and document rather than ask, run tests after every change, hand over versioned files)
9. **Environment to rebuild** (see below)
10. **Files to bring** (the table from step 4)
11. **Open questions for the user** (anything Claude was unsure about from earlier in the chat; each one phrased so the user can answer in a line)
12. **Suggested first message**

## Environment to rebuild

The working environment does not carry over between chats. Write down:

- How to unpack the files and where the project folder ends up
- The exact commands to build, run and test, and which ones need installs (packages, browsers, tools) that must be redone
- What the expected result is when it works (for example "97 + 150 + 146 checks pass"), so the new chat can confirm the starting state
- Limits that bit this chat: command time limits, blocked network domains, tools that were unavailable and so left untested

## If unsure

If Claude is unsure about something settled earlier, say so plainly, put it under "Open questions for the user" in the handover, and suggest a fresh start rather than guessing. Never fill a gap in the handover with something that sounds likely.

## Before finishing, check

- Could a fresh Claude continue from this alone?
- Does every file the new chat needs exist as a downloadable file right now?
- Are the unverified things listed as clearly as the verified ones?
- Are superseded files named, with the instruction to remove them?
