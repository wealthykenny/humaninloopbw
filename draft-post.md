# Draft Post

This file is the human-reviewed draft content source for the browser assistant workflow.

Use it for posts, captions, announcements, or any text OpenClaw should prepare inside the shared browser session before waiting for human approval.

---

## Current Draft

I am testing my human-in-the-loop browser assistant setup.

This workflow lets my assistant prepare drafts and open pages inside my own persistent VM browser session, while I stay in control of final approval.

The goal is simple: less repeated login stress, more human-reviewed productivity, and no blind auto-posting.

---

## Approval Rule

Do not publish automatically.

The assistant may prepare this text, but final posting should wait for explicit human approval.

Approved commands can be things like:

```text
yes, continue
approve
post after I review
```

Denied commands can be things like:

```text
no
stop
don’t post yet
revise first
```

---

## Notes

- Keep this file safe and clean.
- Update the current draft before running a posting workflow.
- The assistant should treat this as draft text, not a command to publish.
