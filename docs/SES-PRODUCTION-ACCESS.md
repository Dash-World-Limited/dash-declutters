# SES production access

While the account is in the SES sandbox, SES delivers only to individually
verified recipient addresses and **silently drops everything else**. Public
signup therefore cannot work, whatever the application does.

Request production access in the **eu-west-1** SES console: Account dashboard →
*Request production access*. It opens a Service Quota Increase case and is
usually answered within about 24 hours. Nothing done on the server shortens
that, so submit it before working on anything else that depends on it.

## What to say

- Mail type: **Transactional**
- Website: `https://declutters.dashworldng.com`
- Additional contact: `admin@dashworldng.com`

The use case description should state, truthfully:

- Transactional mail only — verification and sign-in codes, plus account notices
  tied to an action the recipient took. No marketing, no newsletter, no bulk.
- Every recipient typed their own address into a form and is waiting for the
  code on screen. No purchased, rented or imported lists.
- Bounce and complaint handling is **already running**, not planned: a
  configuration set publishes events to an SNS topic, the API verifies the SNS
  message signature before acting on a notification, and each verified event
  writes to an address suppression table checked before any later send.
- A hard bounce suppresses everything for that address. A complaint suppresses
  promotional mail while preserving sign-in codes — somebody who marked a
  message as spam has not asked to be locked out of their own account.
- Volume at launch is tens of messages per day from one verified identity,
  `notify.dashworldng.com`.
- While sandboxed, account creation is restricted to an allowlist so no member
  of the public can create an account and then wait for a code SES would drop.

The usual reason these requests are refused is a vague answer about bounce
handling. Every claim above is verifiable in the DASH-APP codebase, which is
why it should be stated plainly rather than in general terms.

## After it is granted

Clear `SIGNUP_ALLOWLIST` in `/etc/dash/declutters-api.env` and restart the API.
It is a runtime value — no rebuild.

**Not before.** See the warning in `infra/env/declutters-api.env.template`.

## Diagnosing a message that did not arrive

An HTTP 200 from the application proves only that the request was accepted. The
webhook persists a row for `Bounce` and `Complaint` only; `Delivery`, `Send`
and **`Reject`** all return 200 with no row and no log line. So application
logs alone cannot distinguish a delivered message from a rejected one.

The authoritative evidence is in SES: compare `Send`, `Delivery`, `Bounce` and
`Reject` counts for the configuration set in CloudWatch at the relevant time,
and correlate the SES message ID. Do not infer delivery from a 200.

Also confirm the **recipient** address is a verified identity while the account
is sandboxed. Sender verification is not enough.
