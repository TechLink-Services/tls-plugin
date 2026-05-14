---
name: draft-email-mailto
description: >
  Use this skill whenever the user asks Claude to draft, compose, write, or send an email —
  especially when the Microsoft 365 connector is read-only and cannot send directly. Trigger
  phrases include: "draft an email to", "write an email to", "compose an email", "send an
  email to", "email [person] about", "help me email", "write a message to", "can you draft",
  "put together an email". Always use this skill when the user wants to reach someone via
  email, even if they don't use the word "draft." This skill delivers the finished email as a
  clickable "Open in Outlook" button that pre-fills the recipient, subject, and body.
metadata:
  version: "0.1.0"
---

When this skill triggers, compose a ready-to-send email and deliver it as a `mailto:` link rendered as a markdown button labeled **"Open in Outlook"**. Clicking it opens Outlook with all fields pre-filled. The user can review, edit, save to Drafts, or send directly.

## Step 1 — Find the recipient's email address

If you don't already know the recipient's email, search for it using the Outlook email search tool. Look for recent emails to or from that person by name.

If no email address can be found, ask the user to provide it before proceeding.

## Step 2 — Write the email content

Draft a subject line and body appropriate to the request. Write in plain text — mailto links don't support HTML formatting or attachments.

Keep the body natural and professional. Use the user's name (Ben) to sign off unless they specify otherwise.

## Step 3 — Build the mailto URL

Construct the link using this format:

```
mailto:{to}?subject={subject}&body={body}
```

All three values must be percent-encoded. Key encodings:

| Character | Encoded |
|-----------|---------|
| Space     | `%20`   |
| Newline   | `%0A`   |
| Comma (,) | `%2C`   |
| Em dash (—) | `%E2%80%94` |
| Apostrophe (') | `%27` |
| Plus (+)  | `%2B`   |
| Colon (:) | `%3A`   |
| Forward slash (/) | `%2F` |

Double-check that every space, line break, and special character in the subject and body is encoded — a single unencoded character can break the link.

## Step 4 — Output the button

Present the mailto link as a markdown button:

```markdown
[Open in Outlook](mailto:recipient@example.com?subject=Encoded%20Subject&body=Encoded%20body%20text)
```

Before the button, show the email content in plain text so the user can review it without clicking.

## Output format

```
**To:** recipient@example.com
**Subject:** Subject line here

Body of the email here.

[Open in Outlook](mailto:...)
```

## Notes

- The mailto link does **not** automatically save to Drafts — the user must click Send or save manually from Outlook.
- Mailto links don't support CC, BCC, attachments, or HTML. For those needs, the user should use Outlook directly or request upgraded connector permissions (`Mail.ReadWrite`).
- If the user provides corrections to the draft, rebuild the full mailto URL with the updated content.
