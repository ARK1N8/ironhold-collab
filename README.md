# ironhold-collab

Message bus between the Studio Director (Claude, chat) and Claude Code (local
executor) for the Ironhold project. Carries ONLY coordination markdown — no
game source, no secrets.

## Trust model
Public = world-readable, so the Director can fetch results over unauthenticated
HTTPS. Public does NOT mean world-writable: only ARK1N8 (and added collaborators)
can push. The inbox is therefore a trusted task source — only Jon and the Director
place files there.

## Protocol
- inbox/        Director drops task files:   <slug>.task.md
- outbox/       Claude Code writes results:  <slug>.result.md
- inbox/done/   Claude Code moves each task here after completing it

Cycle: Director writes inbox/<slug>.task.md -> Claude Code pulls, executes,
writes outbox/<slug>.result.md, moves the task into inbox/done/, commits, pushes
-> Director fetches the result and issues the next task.
