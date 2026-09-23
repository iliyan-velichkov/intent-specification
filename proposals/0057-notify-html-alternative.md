# The notify block may carry an HTML alternative: `html`

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7488
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/issues/7488 (tracking issue; no PR yet)
- **Discussion:** (this PR)

## The problem

A notify block sends `body` as plain text, and nothing else can be said about the message's
form. That is right for "invoice 4711 has been approved" and wrong for the messages an
organisation sends to people outside it: a welcome, a confirmation, an invitation. Those are
expected to look like the organisation - a heading, a button, a footer - and every mail client a
recipient is likely to use renders HTML.

The realistic case is a service that provisions a workspace for a customer and mails the person
who asked for it. The declared block says everything the message has to say:

```yaml
notify:
  to: requestedByEmail
  subject: "Your {Application.name} workspace is ready"
  body: |
    Hello,

    Your {Application.name} workspace for {Tenant.name} has been provisioned and is ready to use.

      Organization:  {Tenant.name}
      Application:   {Application.name}
      Sign in at:    {Application.baseUrl}

    Use the link above to access the application and start working with it.
```

What it cannot say is *how*. The only way to a styled message today is to replace the declared
step with hand-written code - which gives up exactly what the block was introduced for: a send
that is part of the model, with the model's resilience and its placeholder resolution, and
nothing to maintain beside it.

## The proposed shape

One optional key, a sibling of `body`, holding the HTML form of the same message:

```yaml
notify:
  to: requestedByEmail
  subject: "Your {Application.name} workspace is ready"
  body: |
    Hello,

    Your {Application.name} workspace for {Tenant.name} has been provisioned and is ready to use.
    Sign in at {Application.baseUrl}
  html: |
    <p>Hello,</p>
    <p>Your <strong>{Application.name}</strong> workspace for <strong>{Tenant.name}</strong>
       has been provisioned and is ready to use.</p>
    <table cellpadding="0" cellspacing="0" style="margin:16px 0">
      <tr><td style="background:#1f5fbf;border-radius:4px;padding:10px 18px">
        <a href="{Application.baseUrl}" style="color:#fff;text-decoration:none">Open the workspace</a>
      </td></tr>
    </table>
```

`body` stays what it is, and stays required. `html` is the same message, marked up; the
placeholders are the ones `body` already resolves - fields, one-hop `Relation.field`, the
link placeholders, the row inside a `forEach` - and nothing new.

## Expected behaviour

A block with `html` sends **one message with two forms of the same text**: the plain form from
`body` and the marked-up form from `html`. A recipient's client shows whichever it can render
best; a client that cannot show HTML shows `body`. Attachments (`attach: print`,
`recordPrint`) ride along unchanged.

The two forms are resolved against the same record at the same moment, so they never disagree
on a value. Every interpolated value in `html` is HTML-escaped by the implementation: a tenant
named `Smith & Sons <Ltd>` renders as that text, never as markup. The author's own markup is
the author's and is sent as written.

## Edge rules

- `html` is optional. `body` remains required: a message MUST always have a plain-text form.
  Declaring `html` without `body` is an error naming the missing key, not a message with an
  empty text part.
- `html` MUST be a non-blank string. Whitespace-only is an error.
- Placeholder resolution in `html` is exactly that of `body`: the same paths, the same
  rejection of multi-hop paths, the same `{recordUrl}` / `{inboxUrl}` / `{appUrl}`, the same
  row scoping inside `forEach`. There is no second resolver and no second set of names.
- The implementation MUST escape the *values* it interpolates into `html` (`&`, `<`, `>`, `"`,
  `'`) and MUST NOT alter the authored markup around them. An `href="{appUrl}"` therefore
  receives a correctly escaped attribute value.
- `subject` is text and takes no markup; this proposal does not change it.
- A delivery channel that cannot carry markup (a chat channel, an in-app notice) MUST deliver
  `body` and MUST ignore `html`; declaring `html` for such a channel is not an error, because
  the same block may reach several channels.
- The failure semantics of the call site are unchanged: the message is one send, and it
  succeeds or fails as one.

## Prior art / workarounds

- **Hand-written send.** Replace the declared step with a delegate that builds the HTML and
  calls the platform's mail API with a text and an HTML part. It works, and it is what the
  problem scenario is about to do. It also removes the send from the model: the resilience
  keys, the placeholder validation at parse time, and the "nobody to mail is a no-op" rule all
  have to be re-implemented by hand, and the message's text lives in code rather than beside
  the entities it describes.
- **A template file** (`template: mail/workspace-ready.html`) next to the print templates.
  Considered and not proposed here: it introduces a new file kind, a lookup rule, and a
  second interpolation surface with its own escaping story, for a gain (sharing one layout
  across blocks) that a later proposal can add on top of `html` without changing it. The inline
  form is the smallest shape that closes the gap; block scalars keep it readable.
- **Markdown in `body`** rendered to HTML by the implementation. Rejected: it changes the meaning
  of every existing `body` (a `*` or a `#` in today's text would start to render), and the
  author still has no control over the result.
- Other formats: a mail template in most application frameworks is a text/HTML pair with
  escaped interpolation - the shape proposed here, and the one recipients' clients were built
  for (`multipart/alternative`).

## Specification text

**Anchor:** Declarative glue > The notify block - and `attach: print`, sending the document itself
(a new subsection following *Links back to the application*)

#### The message's marked-up form: `html`

A notify block sends `body` as plain text. A message meant for someone outside the
organisation - a welcome, a confirmation, an invitation - is expected to look like the
organisation, so the block MAY also carry the same message marked up:

```yaml
    notify:
      to: requestedByEmail
      subject: "Your {Application.name} workspace is ready"
      body: "Your {Application.name} workspace for {Tenant.name} is ready. Sign in at {Application.baseUrl}"
      html: |
        <p>Your <strong>{Application.name}</strong> workspace for <strong>{Tenant.name}</strong> is ready.</p>
        <p><a href="{Application.baseUrl}">Open the workspace</a></p>
```

`html` is the marked-up form of `body`, not a second message: the same recipient, the same
subject, the same attachments, the same placeholders resolved against the same record.

> **Normative.** `html` is OPTIONAL and `body` remains REQUIRED: a message MUST always have a
> plain-text form, and a block declaring `html` without `body` MUST be rejected. `html` MUST be a
> non-blank string. Every `{placeholder}` in `html` resolves by the rules of `body` - the same
> paths, the same reserved link placeholders, the same row scoping inside a `forEach` - and an
> implementation MUST NOT introduce names or paths that `body` does not accept.

> **Normative.** An implementation MUST send the two forms as alternatives of ONE message, so a
> recipient's client renders the marked-up form when it can and the plain form otherwise. It MUST
> escape every interpolated VALUE in `html` as HTML text and MUST NOT alter the authored markup
> around it. A delivery channel that cannot carry markup MUST deliver `body` and ignore `html`.

Why the values are escaped and the markup is not: the markup is the author's design and the
values are somebody's data. A name containing `<` is a name, and a message that let it become an
element would let a record rewrite the message.

**Anchor:** Appendix A: DSL index

| [`notify.html`](#the-messages-marked-up-form-html) | the same message marked up - sent as the alternative form of `body`, values escaped, placeholders unchanged |

## DSL index

| Construct | What it does |
| --- | --- |
| `notify.html` | the marked-up form of a notify block's `body`, sent as an alternative of the same message |
