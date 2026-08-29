# Sending via himalaya (v2.1.0) — config & gotchas

The bundled `himalaya` skill's config block is the **pre-v1.2.0 format** and is
**wrong for himalaya v2.1.0+**. This file is the v2.1.0 reference that the
`email-report-delivery` skill relies on. If you get `No backend matching 'auto'
is configured`, you are using the old format.

## 1. Install
```bash
curl -sSL https://raw.githubusercontent.com/pimalaya/himalaya/master/install.sh | PREFIX=~/.local sh
# binary at ~/.local/bin/himalaya ; verify:  himalaya --version
```

## 2. Store the password (NOT in .env — it is secret-guarded)
```bash
mkdir -p ~/.local/creds
printf '%s' 'THE_PASSWORD' > ~/.local/creds/email
chmod 600 ~/.local/creds/email
```
Feed it to himalaya via `password.command = "cat /home/you/.local/creds/email"`.

## 3. config.toml — v2.1.0 format (URIs, sasl.login, mailbox.alias)
```toml
[accounts.myname]
email = "you@example.com"
display-name = "Your Name"
default = true

imap.server = "imaps://imap.example.com:993"          # imaps:// = implicit TLS
imap.sasl.login.username = "you@example.com"
imap.sasl.login.password.command = "cat /home/you/.local/creds/email"

smtp.server = "smtps://smtp.example.com:465"          # smtps:// = implicit TLS (465)
smtp.sasl.login.username = "you@example.com"
smtp.sasl.login.password.command = "cat /home/you/.local/creds/email"

mailbox.alias.inbox = "INBOX"
mailbox.alias.sent  = "INBOX.Sent Items"   # REAL names — run `himalaya mailbox list`
mailbox.alias.drafts = "INBOX.Drafts"
mailbox.alias.trash  = "INBOX.Trash"
```
Old (rejected) keys: `backend.type/host/port/login`, `backend.auth`,
`folder.aliases.*`. New keys: `imap.server`/`smtp.server` (URI),
`sasl.<mech>.username/.password.command`, `mailbox.alias.*`.

## 4. AUTH mechanism — probe the server, don't guess
The server may reject `sasl.plain` with
`SMTP AUTH PLAIN failed: rejected 504 PLAIN authentication mechanism not supported`.
Probe the advertised mechanisms and match `sasl.<mech>` to it:
```bash
printf "EHLO x\r\nQUIT\r\n" | openssl s_client -connect HOST:465 -quiet 2>/dev/null | grep AUTH
# e.g. "250-AUTH LOGIN"  ->  use  sasl.login.username / sasl.login.password.command
# (not sasl.plain.*)
```

## 5. Commands that changed in v2.1.0
| Old (pre-v1.2.0) | v2.1.0 |
|---|---|
| `himalaya folder list` | `himalaya mailbox list` |
| `envelope list --limit N` | `envelope list --page-size N` |
| `--folder "Sent"` | `-m "INBOX.Sent Items"` (alias-resolved) |

## 6. Send a pre-built MIME message (HTML body)
Build the message with `MIMEMultipart` (see the main SKILL.md), write it to a file,
then pipe it — himalaya reads RFC 5322 from stdin:
```bash
himalaya message send --save "INBOX.Sent Items" < message.txt
# verify:
himalaya envelope list -m "INBOX.Sent Items" --page-size 3
```
`--save` uses the `mailbox.alias.sent` mapping. Confirm the message appears in the
Sent folder before reporting success.
