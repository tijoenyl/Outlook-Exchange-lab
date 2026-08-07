# Microsoft 365 / Outlook and Exchange Online Troubleshooting Lab

Ten simulated Tier 1 support tickets worked end to end in a live Microsoft 365 tenant. Each one covers a request or fault I'd expect to see on a service desk queue: shared mailbox access, calendar delegation, distribution lists, missing mail, quarantine, onboarding, forwarding, and Outlook client issues.

Every ticket follows the same shape — reproduce the problem first, find the cause, fix it, then verify from the user's side.

## Environment

- Tenant: `lainecorp.onmicrosoft.com`
- Licensing: Enterprise Mobility + Security E5, Office 365 E5
- Admin portals: Microsoft 365 admin center, Exchange admin center, Microsoft Defender
- Clients: Outlook on the web, classic Outlook desktop (Windows 11)
- Test users: Kyle Lowry, Jason Bourne, Kennedy Ryan, Sharon Love, Marcus Reed
- External sender: a personal Gmail account, used to test anything that crosses the org boundary

## Tickets

| # | Ticket | Area |
|---|---|---|
| 01 | [Shared mailbox access request](#ticket-01--shared-mailbox-access-request) | Permissions |
| 02 | [Calendar permissions and delegate access](#ticket-02--calendar-permissions-and-delegate-access) | Permissions |
| 03 | [Distribution list membership](#ticket-03--distribution-list-membership) | Mail flow |
| 04 | [Missing emails](#ticket-04--missing-emails) | Mail flow |
| 05 | [Oversized Outlook data file](#ticket-05--oversized-outlook-data-file) | Client |
| 06 | [New employee onboarding](#ticket-06--new-employee-onboarding) | Provisioning |
| 07 | [Quarantined message release](#ticket-07--quarantined-message-release) | Security |
| 08 | [Email forwarding request](#ticket-08--email-forwarding-request) | Mail flow |
| 09 | [Mail stuck in Outbox](#ticket-09--mail-stuck-in-outbox) | Client |
| 10 | [Automatic replies not reaching external senders](#ticket-10--automatic-replies-not-reaching-external-senders) | Client |

---

## Ticket 01 — Shared Mailbox Access Request

**Requester:** Kennedy Ryan
**Resource:** HR Department (hrdepartment@lainecorp.onmicrosoft.com)
**Tools:** M365 admin center, Exchange admin center, Outlook on the web

**Issue:** HR wanted a single address for employee inquiries instead of mail going to individual mailboxes. Kennedy Ryan needed to read the mailbox and send from it.

### Creating the mailbox

Built the shared mailbox in the M365 admin center under Teams & groups → Shared mailboxes, named HR Department with the alias `hrdepartment`. Shared mailboxes under 50 GB don't need a license, so this is the right object for a departmental address rather than a licensed user account.

![Creating the shared mailbox](screenshots/t1-01.png)
![Mailbox created](screenshots/t1-02.png)
![Delegation tab showing no permissions assigned](screenshots/t1-03.png)

### Reproducing the failure

Signed in as Kennedy and used Open another mailbox. The mailbox failed to load:

```
BootResult: accessDenied
err: Microsoft.Exchange.Data.StoreObjects.AccessDeniedException
st: 500
```

The address resolved, so the mailbox exists and the alias is right. That narrows it to delegation.

![Open another mailbox](screenshots/t1-04.png)
![AccessDeniedException](screenshots/t1-05.png)

### Granting Full Access

EAC → HR Department → Mailbox delegation → Read and manage (Full Access) → Edit. Added Kennedy and saved. EAC warned the change can take up to 5 minutes to apply.

![Adding Read and manage permission](screenshots/t1-06.png)
![Permission confirmed](screenshots/t1-07.png)
![Mailbox now opens](screenshots/t1-08.png)

### Send failure

Kennedy could read the mailbox but composing from it and hitting Send returned "You don't have permission to send messages from this mailbox."

Full Access and Send As are separate. Full Access opens the mailbox; it doesn't authorize outbound mail on the mailbox's behalf. Granting only one is what turns this into a second ticket.

![Send blocked](screenshots/t1-09.png)

### Granting Send As

Same delegation panel, Send as → Edit. Added Kennedy and confirmed the entry landed in the list.

Worth checking with the requester which they actually want here. Send As rewrites the From address to HR Department. Send on Behalf would show "Kennedy Ryan on behalf of HR Department" instead.

![Adding Send As](screenshots/t1-10.png)
![Delegation list](screenshots/t1-11.png)

### Verification

Sent a test message from the HR mailbox to Sharon Love. It went through and shows in Sent Items with HR Department as the sender.

![Sent message](screenshots/t1-12.png)

**Resolution:** Created the shared mailbox, granted Full Access and Send As to Kennedy Ryan, confirmed both read and send work.

---

## Ticket 02 — Calendar Permissions and Delegate Access

**Requester:** Jason Bourne
**Calendar owner:** Kyle Lowry
**Tools:** Outlook on the web, Exchange admin center

**Issue:** Jason could see Kyle's calendar but only as blocks marked Busy. He needed to see what the meetings actually were, and later needed to book on Kyle's behalf.

### Confirming what he could see

Added Kyle's calendar to Jason's view. Everything showed as Busy with no subject or location, which is the default free/busy level for people inside the org.

![Kyle's calendar showing only Busy blocks](screenshots/t2-01.png)

Checked the mailbox in EAC first to rule out anything set at the delegation level. Send As, Send on Behalf and Full Access were all empty, so this was a calendar folder permission rather than a mailbox one.

![EAC mailbox delegation](screenshots/t2-02.png)

### Raising the view level

From Kyle's calendar, Sharing and permissions. There are five tiers here: Not shared, Can view when I'm busy, Can view titles and locations, Can view all details, Can edit. Jason was sitting at the org-wide default. Set him to Can view all details.

![Sharing and permissions with the access tiers](screenshots/t2-03.png)

Back in Jason's view, meeting subjects now render instead of Busy.

![Meeting titles now visible](screenshots/t2-04.png)

### Escalating to delegate

Kyle followed up asking Jason to be able to accept meeting invites for him. That's past a permission tier — it needs a delegate.

Added Jason under Delegates. A delegate can create, modify and delete items and respond to invitations on the owner's behalf. The Send invitations and responses to setting controls where meeting traffic lands; left it on Delegate only so invites route to Jason rather than cluttering Kyle's inbox. Private events and category management stay off unless the owner asks for them.

![Delegate configuration](screenshots/t2-05.png)

**Resolution:** Raised Jason to Can view all details, then added him as a delegate with invitations routed to him only.

---

## Ticket 03 — Distribution List Membership

**Requester:** Kyle Lowry
**Group:** Sales Team (salesteam@lainecorp.onmicrosoft.com)
**Tools:** Exchange admin center, Outlook on the web, external Gmail

**Issue:** Kyle reported he wasn't getting Sales Team email that colleagues were receiving.

### Confirming the gap

A test message to the Sales Team group landed in Jason's inbox.

![Jason receives the group message](screenshots/t3-01.png)

Nothing in Kyle's.

![Kyle's inbox is empty](screenshots/t3-02.png)

Both users are fine individually, so the mailbox isn't the problem — it's the group. EAC → Groups → Distribution list → Sales Team showed one owner and one member, and the member was Jason. Kyle was never added.

![Sales Team with a single member](screenshots/t3-03.png)

### Adding the member

Added Kyle under Members and saved.

![Kyle added to the group](screenshots/t3-04.png)

A second test to the group reached him.

![Kyle receives the group message](screenshots/t3-05.png)

### Second issue: external senders

Kyle then mentioned a client emailing the Sales Team address was getting bounces. Sent a message from an external Gmail account to reproduce it.

![Sending to the group from outside the org](screenshots/t3-06.png)

Distribution lists default to closed — only senders inside the organization can post. That's a sensible default for internal lists, but wrong for an address clients are expected to use. Changed it under Delivery management to allow inside and outside.

![Delivery management set to allow external senders](screenshots/t3-07.png)

The external message came through.

![External message delivered](screenshots/t3-08.png)

**Resolution:** Added Kyle to the Sales Team list and opened the group to external senders.

---

## Ticket 04 — Missing Emails

**Affected user:** Kyle Lowry
**Sender:** Jason Bourne
**Tools:** Exchange admin center message trace, Outlook on the web

**Issue:** Kyle said messages from Jason weren't arriving. Jason confirmed he'd sent three that morning.

### Message trace

Ran a trace in EAC filtered to Jason as sender and Kyle as recipient.

![New message trace](screenshots/t4-01.png)

All three showed Delivered, so this was never a mail flow problem — the messages reached the mailbox.

![Three messages delivered](screenshots/t4-02.png)

Opening the detail on one gave the actual answer:

> The message was delivered to the recipient's mailbox. Because of an Inbox rule the recipient set up, the message was delivered to the following folder: **Deleted Items**

![Trace detail naming Deleted Items and an inbox rule](screenshots/t4-03.png)

This is why trace is the first stop on a missing mail ticket instead of the client. Delivered plus a folder name saves you digging around in Outlook.

### Confirming in the mailbox

Searched Kyle's mailbox across all folders for mail from Jason. All three were sitting in Deleted Items.

![Messages found in Deleted Items](screenshots/t4-04.png)
![Message opened](screenshots/t4-05.png)

### Finding the rule

Settings → Mail → Rules turned up a rule named Cleanup.

![Rules list](screenshots/t4-06.png)

Expanded: if the message was received from Jason Bourne, delete it and stop processing further rules. Almost certainly created by accident, or set up for something temporary and never removed.

![Rule condition and action](screenshots/t4-07.png)

Deleted the rule.

![Deleting the rule](screenshots/t4-08.png)

### Recovery and verification

Selected the three messages and moved them back to the Inbox.

![Moving messages back to Inbox](screenshots/t4-09.png)
![Messages restored](screenshots/t4-10.png)

Had Jason send a fresh test. It landed in the Inbox normally.

![Test message sent](screenshots/t4-11.png)
![Test message delivered to Inbox](screenshots/t4-12.png)

If the messages had already been purged from Deleted Items, the next step would have been Recover deleted items, which pulls from the Recoverable Items folder for 14 days by default.

**Resolution:** Removed the inbox rule that was deleting Jason's mail and restored the affected messages.

---

## Ticket 05 — Oversized Outlook Data File

**Affected user:** Kyle Lowry
**Tools:** File Explorer, classic Outlook desktop, Exchange admin center

**Issue:** User reported slow Outlook performance and a large local data file.

### Checking the file

Located the OST under `C:\Users\<user>\AppData\Local\Microsoft\Outlook`.

![OST file size](screenshots/t5-01.png)

The OST is a local cache of the mailbox, not the mailbox itself. Deleting it doesn't lose mail — it rebuilds on next sync — but on a large mailbox that rebuild is slow, so it's worth trying the sync window first.

### Reducing the sync window

File → Account Settings → Account Settings → Change → Cached Exchange Mode. The slider was set to All, meaning the entire mailbox history was cached locally.

![Cached Exchange Mode set to All](screenshots/t5-02.png)

Dropped it to 3 months. Older mail stays in the mailbox and remains reachable, it just gets pulled on demand instead of stored on disk.

![Slider set to 3 months](screenshots/t5-03.png)

The file size didn't move after the change. This lab mailbox only holds a handful of messages, so there was nothing outside the three month window to drop — the OST is already at its floor. On a real mailbox with years of mail this is where the reduction shows up.

![File size after the change](screenshots/t5-04.png)

### Online archive

For a user who genuinely needs the history but not on their laptop, the other lever is an online archive. Enabled it on Kyle's mailbox in EAC.

![Archive enabled in EAC](screenshots/t5-05.png)

It appears in Outlook as a separate mailbox in the folder tree. Archived mail lives server side and is searchable, but doesn't count against the local cache.

![Online archive in the Outlook folder tree](screenshots/t5-06.png)

**Resolution:** Reduced the cached sync window to 3 months and enabled an online archive. Noted the file size result above is a limitation of the lab mailbox, not the procedure.

---

## Ticket 06 — New Employee Onboarding

**New hire:** Marcus Reed, Sales
**Tools:** M365 admin center, Exchange admin center

**Issue:** New sales hire starting, needs an account, mailbox, group access, and MFA registered before day one.

### Creating the account

M365 admin center → Users → Active users → Add a user. Marcus Reed, username `mreed`, auto-generated password with a forced change on first sign-in.

![Creating the user](screenshots/t6-01.png)

Assigned Enterprise Mobility + Security E5 and Office 365 E5.

![User created with licenses](screenshots/t6-02.png)
![License assignment confirmed](screenshots/t6-03.png)

### Mailbox

The Exchange Online mailbox provisions off the Office 365 license rather than being created separately. Confirmed it appeared in EAC as a UserMailbox before moving on — worth checking rather than assuming, since it isn't always instant.

![Mailbox provisioned](screenshots/t6-04.png)

### Group and mailbox access

Added Marcus to the Sales Team distribution list so he receives team mail.

![Added to Sales Team](screenshots/t6-05.png)

Granted Read and manage on the Sales Department shared mailbox, matching what the rest of the team has.

![Shared mailbox access granted](screenshots/t6-06.png)

### MFA registration

Walked the first sign-in as Marcus. Registration prompted for Microsoft Authenticator, then a second method.

![Authenticator setup](screenshots/t6-07.png)
![QR code registration](screenshots/t6-08.png)
![Registration complete with two methods](screenshots/t6-09.png)

### Verification

Signed in to Outlook as Marcus. Mailbox loads, correct identity.

![First sign-in](screenshots/t6-10.png)

**Resolution:** Account created and licensed, mailbox confirmed, distribution list and shared mailbox access granted, MFA registered with two methods.

---

## Ticket 07 — Quarantined Message Release

**Affected user:** Kyle Lowry
**Sender:** external Gmail address
**Tools:** Exchange admin center, Microsoft Defender, Outlook on the web

**Issue:** Kyle was expecting an invoice from an external contact and it never arrived.

### Ruling out the obvious

Checked the Inbox — not there.

![Inbox](screenshots/t7-01.png)

Checked Junk Email — also empty. That already tells you it isn't a junk filter issue at the mailbox level, so it was stopped before delivery.

![Junk Email is empty](screenshots/t7-02.png)

### Message trace

Ran a trace against the external sender and Kyle.

![Message trace](screenshots/t7-03.png)

Status came back Received and Processed but **Not yet delivered**, with the detail explaining the message was identified as spam and routed to quarantine by an anti-spam policy.

![Trace detail showing the quarantine verdict](screenshots/t7-04.png)

### Release

Defender → Review → Quarantine. The message was there, flagged High Confidence Phish under the anti-spam policy, release status Needs review.

![Message in quarantine](screenshots/t7-05.png)

Released it to the recipient inbox.

![Released](screenshots/t7-06.png)

### Preventing a repeat

Releasing one message doesn't stop the next one being caught. Added the sender to the Tenant Allow/Block List with a 45 day expiry after last use.

Worth being deliberate about this step rather than reflexive — an allow entry weakens filtering for that sender, so it should only go in for a known, verified contact. Not for anything that looked like phishing on inspection.

![Allow entry added](screenshots/t7-07.png)

### Verification

The released message appeared in Kyle's inbox. The body is the GTUBE test string, which is what triggered the spam verdict in the first place.

![Released message delivered](screenshots/t7-08.png)

A follow-up message from the same sender delivered straight to the inbox with no quarantine.

![Follow-up message delivers normally](screenshots/t7-09.png)

**Resolution:** Released the quarantined message and added the sender to the tenant allow list.

---

## Ticket 08 — Email Forwarding Request

**Affected user:** Kyle Lowry
**Forward to:** Kennedy Ryan
**Tools:** Exchange admin center, Outlook on the web

**Issue:** Kyle going on leave, wants his mail routed to Kennedy while he's out.

### Configuring the forward

EAC → Mailboxes → Kyle Lowry → Manage email forwarding. Turned forwarding on.

The panel warns about automatic forwarding to external recipients, which is worth reading rather than clicking past — external auto-forwarding is a common exfiltration path after an account compromise, which is why it's restricted by default. This one is internal, so it doesn't apply here.

![Forwarding enabled](screenshots/t8-01.png)

Selected Kennedy Ryan as an internal recipient.

![Selecting the internal recipient](screenshots/t8-02.png)
![Kennedy set as the forwarding address](screenshots/t8-03.png)
![Settings saved](screenshots/t8-04.png)

One thing worth confirming with the user before saving: whether to keep a copy in the original mailbox. Without it, Kyle comes back from leave to an empty inbox and no record of what came in.

Note also that internal forwarding set at the mailbox level can't be changed by the mailbox owner — only an admin can remove it. That matters when the leave ends and someone needs to turn it off.

### Verification

Sent a test to Kyle. It arrived in Kennedy's inbox, still addressed to Kyle.

![Forwarded message in Kennedy's inbox](screenshots/t8-05.png)

**Resolution:** Mailbox-level forward configured from Kyle to Kennedy and verified.

---

## Ticket 09 — Mail Stuck in Outbox

**Affected user:** Kyle Lowry
**Client:** classic Outlook desktop
**Tools:** Outlook desktop

**Issue:** User reported messages sitting in the Outbox and not sending. No error on send.

### Baseline

Status bar read Connected to: Microsoft Exchange, so I had a known good state to compare against.

![Connected](screenshots/t9-01.png)

### Reproducing it

Toggled Send / Receive → Work Offline. Status bar changed to Working Offline.

![Working Offline](screenshots/t9-02.png)

Composed a message and sent it.

![Composing a message](screenshots/t9-03.png)

It dropped into the Outbox and stayed there. This is the exact symptom users describe, and there's no error dialog — Outlook accepts the send and just queues it.

![Message sitting in the Outbox](screenshots/t9-04.png)

Send / Receive → Show Progress → Outlook Connection Status confirmed it: every connection listed as Offline, including the archive and the shared mailbox.

![All connections offline](screenshots/t9-05.png)

The tell here is the status bar. If it says Working Offline, the problem is the client's own toggle rather than the network or the server, and no amount of restarting Outlook fixes it. Users hit this by clicking Work Offline without realizing what it does.

### Fix and verification

Toggled Work Offline back off. Status bar returned to Connected to: Microsoft Exchange.

![Reconnected](screenshots/t9-06.png)

The queued message flushed out of the Outbox on the next send/receive and appeared in Sent Items.

![Message in Sent Items](screenshots/t9-07.png)

**Resolution:** Work Offline was enabled on the client. Turned it off and the queued mail sent.

---

## Ticket 10 — Automatic Replies Not Reaching External Senders

**Affected user:** Kyle Lowry
**Tools:** Outlook desktop, Exchange admin center, external Gmail

**Issue:** User set an out of office. Colleagues got the reply, clients emailing from outside didn't.

### Setting the reply

File → Automatic Replies. Turned on Send automatic replies with a message on the Inside My Organization tab.

![Automatic replies configured](screenshots/t10-01.png)

### Confirming internal works

Jason sent Kyle a message.

![Internal message sent](screenshots/t10-02.png)

The auto-reply came back.

![Internal auto-reply received](screenshots/t10-03.png)

### Reproducing the failure

Sent to Kyle from an external Gmail account. No reply came back.

![External message sent](screenshots/t10-04.png)

Inside and Outside My Organization are two separate messages with separate toggles. Filling in the inside tab does nothing for external senders — the outside tab has its own enable checkbox, and it's off by default.

Enabled Auto-reply to people outside my organization, set to Anyone outside my organization rather than My Contacts only, and added the message.

![Outside My Organization enabled](screenshots/t10-05.png)

The same setting is visible admin-side in EAC under Manage automatic replies, which is where you'd check it if the user can't get into their client.

![Automatic replies in EAC](screenshots/t10-06.png)

### Verification

Sent again from the external account. The auto-reply came through.

![External auto-reply received](screenshots/t10-07.png)

**Resolution:** The external auto-reply option was disabled. Enabled it, scoped to anyone outside the organization, and confirmed with an external test.
