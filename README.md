# patch-outreach-automation

Vulnerability Remediation Outreach Pipeline

An automated pipeline that closes the loop between a vulnerability scanner's patch-gap report and an actual patched machine — built during a cybersecurity internship at a healthcare-sector organization to replace hours of manual triage-and-contact work with a few clicks.

Goal: eliminate the manual bottleneck between "here's a list of vulnerable machines" and "the right person actually got contacted and booked a fix" — end-to-end, with accountability at every step.

Architecture
Vulnerability Scanner   →   VBA Macro         →   Deduplicated      →   Power Automate   →   Teams Adaptive Card   →   Recap Email
   CSV Export                (identifies           Campaign List         Flow (send +          + Bookings Link          (sent/failed
 (assets missing              primary user                                track outcomes)        (self-service            tally to
  security patches)           from login logs)                                                     scheduling)             analyst)

Stack:

Source data: Vulnerability management platform export (list of assets with missing Windows security patches)
User attribution: Excel VBA macro parsing raw Windows login event logs
Dedup / merge: custom VBA tool for combining multi-month campaign lists into one contact list
Orchestration: Microsoft Power Automate
Delivery: Microsoft Teams Adaptive Cards, sent via the Flow bot
Scheduling: Microsoft Bookings
What This Demonstrates
Designing a user-attribution algorithm — most-frequent-logged-in-user over a rolling 90-day window, built and validated against real login logs, rather than a naive "who logged in last" approach that misattributes machines to IT staff doing rotations or reimages
Handling messy real-world data: excluding service/technician accounts, deduplicating people who span multiple monthly reporting cycles, and rolling up multiple machines under one contact instead of sending duplicate outreach
Building a multi-platform automated workflow (Excel → Power Automate → Teams → Bookings) with explicit success/failure tracking rather than a fire-and-forget script
Debugging platform-level quirks — connector bugs, permission limitations, and undocumented trigger behavior — rather than just wiring together a happy-path demo
Scoping work like a stakeholder-facing engineer: writing a problem/success-criteria/scope brief and getting sign-off before building, and keeping incomplete components out of presentation-facing summaries
Build Notes & Troubleshooting

A few of the harder problems along the way — these were more instructive than the parts that just worked on the first try:

"Last login" is the wrong attribution signal. Early versions attributed each machine to whoever logged in most recently. In practice this frequently pointed at IT staff who'd touched the machine for an unrelated reason, not the actual owner. Fixed by switching to a most-frequent-user calculation over a rolling 90-day window — validated by reproducing the logic in Python against real log exports before trusting it in production.

Duplicate outreach across reporting cycles. The scanner exports one list per campaign, so a person whose machine appeared in two consecutive months' reports would get messaged twice. Solved with a dedicated merge tool that combines selected campaign sheets, deduplicates on username, and rolls multiple machine names into a single comma-separated field per person.

Teams won't send 1:1 messages "as the user." Sending an Adaptive Card as the signed-in user (rather than the automation's bot identity) returns a hard BadRequest — Microsoft doesn't support that send path for 1:1 DMs. Fixed by sending all cards as the Flow bot instead.

Excel Online connector silently fails to render table columns. A separate booking-tracker flow needed to write rows into an Excel table, but the standard "Add a row into a table" connector refused to render the table's column fields for certain workbook/table states — a confirmed platform bug, not a config error, after six different diagnostic attempts. Workaround in progress: replacing that connector action with an Office Scripts function instead.

Trigger data doesn't match its documented shape. The Outlook "event created" trigger was expected to return meeting attendees as a structured array; it actually returns a semicolon-delimited string, and self-booking test events don't reliably surface the booker's email in the attendee fields at all. Real (non-self) test bookings are needed to confirm the actual data shape before the extraction logic can be finished.

Power Automate manual triggers can't query live data for a dropdown. Wanted a dropdown of valid campaign names pulled live from the spreadsheet; manual triggers don't support that natively. Resolved by using a static, manually-maintained dropdown instead of fighting the platform.

Sample Run

Scenario: monthly patch-gap export flagging ~40–75 assets as missing a critical Windows security update.

Before: an analyst manually cross-references each machine against login logs, figures out who to contact, and sends individual messages — realistically a multi-hour task, done by hand, with no record of who was or wasn't reached.

After: the analyst runs the macro against the new export, reviews the clean output table, and kicks off the flow. Each identified user gets a Teams card with a one-click scheduling link; the flow tracks who was successfully messaged versus who failed (bad data, missing user, delivery failure) and emails the analyst a recap the moment the run finishes.

The actual leverage point: the attribution step. Anyone can send a mail merge — the harder problem was reliably figuring out who to send it to from raw, noisy login data, without pinging the wrong person or an IT tech's account.
