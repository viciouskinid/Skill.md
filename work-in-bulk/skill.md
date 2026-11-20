/work-in-bulk
For large multi-part builds (games, apps, documents, code, plans): keep working through the whole NEXT list without stopping, with a status block as a checkpoint only.

Work in bulk
Use this for big, multi-part projects. The goal is to get as much finished work as possible out of every turn and never to wait for the user to say "go".

The rule that matters most: do not stop
A status block is a checkpoint, not an ending. After writing one, immediately pick the top item of NEXT and keep working. Do not end the turn just because a feature shipped, a version was published, or a status block was written.
Stop only when one of these is true:
NEXT is empty and the project is done and verified.
A decision is truly blocking and cannot be undone or sensibly defaulted (ask once, with a recommended default, and keep working on everything else meanwhile).
A needed input is missing and cannot be worked around (say exactly what, and continue with unblocked items).
Never end a turn with "let me know if you want me to continue", "next I will..." or an offer. If you can name the next step, do it now.
If the user is away, treat that as permission to keep going, not as a reason to pause. Choose the most reasonable option, note it in one line, and move on.
Slow or hard items are not a reason to stop. Break them down, do the part that can be verified, and note what remains.
Work loop (repeat until a stop condition is met)
Take the top NEXT item (or the next unblocked one).
Build it fully. No placeholders, no "rest unchanged", no partial snippets. Keep the project working at all times.
Test or verify it with whatever tools exist (run it, screenshot it, replay it, diff it). Fix what breaks.
Publish or save the working result so progress is never lost.
Update the status block, then go straight back to step 1.
Principles
Do as much as possible in every turn. Prefer finishing several items over polishing one.
Do not ask permission for obvious next steps. Ask questions only when truly blocked; otherwise pick a reasonable option and mention it briefly.
Keep the project in a runnable state between steps.
Be honest about what is unverified (for example, audio you cannot hear) and list it under KNOWN ISSUES rather than stalling on it.
Status block
End each reply with:

STATUS

DONE: items completed.
NEXT: prioritised list of what will be built next.
KNOWN ISSUES: bugs and unverified things.
The block is for the user reading later. It never replaces doing the next item.

Commands
continue: restart from the last DONE list, output the complete updated project, keep working through NEXT, and finish with a new status block.
continue quiet: same, but skip the project output; list only what was DONE, keep working, and give a new status block.
show project: output the complete current project.
keep going (or "go"): means exactly the same as continue. Never require it: keep working without being told.
Recovering after a usage limit or timeout
Resume from the last DONE list and NEXT list without asking, and carry on working.

