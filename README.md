# Remote Team Culture Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for building remote team culture through intelligent matching, recognition, celebrations, and morale tracking inside Slack and Microsoft Teams.

The Remote Team Culture Platform combines virtual coffee matching, peer recognition, celebrations, and engagement analytics into a single open-source toolkit for distributed teams. It is aimed at HR and People Ops leaders, engineering managers, and DEI leaders at remote-first or hybrid organisations who need to sustain belonging and connection without piecing together half a dozen point solutions.

---

## Why Remote Team Culture Platform?

- Incumbents are fragmented across single-purpose tools — Donut for matching, HeyTaco for recognition, Doozy for celebrations, Culture Amp for surveys — forcing buyers to stitch together multiple subscriptions to cover one cultural surface area.
- Cross-platform parity is rare: Donut and Doozy are Slack-only, while Teams-first organisations are pushed toward weaker alternatives. Hybrid orgs running both chat platforms have no first-class option.
- Analytics in lightweight tools stop at participation counts. None of the reviewed Slack-native products surface relationally isolated employees before disengagement turns into attrition.
- Pricing pressure: lightweight Slack apps charge $1–$3/user/month and full engagement suites (Cooleaf, Culture Amp) use custom enterprise pricing at $5–$10/employee/month, leaving smaller teams without a viable affordable suite.
- Matching algorithms remain mostly random or rule-based; goal-directed and DEI-aware pairing exists in only a handful of premium tools (RandomCoffee, Together).

---

## Key Features

### Matching & Connection

- Configurable matching engine supporting random, rules-based (department, location, tenure), and affinity matching
- 1:1 and small-group pairings delivered inside Slack and Microsoft Teams DMs
- Calendar integration (Google Calendar, Outlook) for one-click scheduling that respects timezones
- Onboarding journeys that pair new hires with buddies or mentors
- Optional live video speed-networking sessions via Zoom or Teams Breakout Rooms

### Recognition & Celebrations

- Peer recognition with points, values tagging, and an optional reward catalogue
- Birthday and work-anniversary celebration automation, with HRIS-driven date sync
- Digital team-signed cards that nudge teammates to contribute before the celebration date
- Watercooler conversation starters delivered into channels

### Engagement Analytics

- Participation analytics dashboard (match accepted, meeting completed, recognition given)
- Pulse survey module with configurable questions and trend dashboards
- Network graph visualisation showing relationship density across teams
- Isolation-risk flagging for employees with declining connection participation
- Optional benchmark comparison against anonymised aggregate platform data

### Enterprise & Integration

- Slack and Microsoft Teams native bots (no separate employee-facing app required)
- HRIS sync for roster and celebration dates (BambooHR, Workday, and similar platforms)
- SCIM provisioning and SAML SSO for enterprise user management
- OAuth 2.0 authentication and GDPR-compliant data handling with consent records and data minimisation
- Optional MCP server exposing engagement metrics to AI assistants and internal dashboards

---

## AI-Native Advantage

AI moves matching beyond randomness — pairing people on complementary skills, project overlap, career goals, and DEI objectives rather than coin flips. Engagement signals are aggregated from participation data, response rates, and async sentiment to produce a real-time team health score without survey fatigue. Anomaly detection identifies sub-teams or individuals becoming relationally isolated within the network graph and flags them before disengagement turns into attrition. A monthly plain-language culture narrative synthesises quantitative metrics with qualitative signals, replacing the manual dashboard-building work currently done by People Ops.

---

## Tech Stack & Deployment

The platform is designed to live where employees already work: Slack Block Kit and Microsoft Teams Adaptive Cards for in-chat interactions, with an admin web portal for HR configuration and analytics. Integration relies on open standards — OAuth 2.0 for chat, calendar, and HRIS connections; SCIM for enterprise user provisioning; and CalDAV / Google Calendar API for scheduling. HRIS sync targets BambooHR and Workday for MVP. Self-hosted and cloud deployment modes are both anticipated, and the project is intended to ship its own engagement survey question set rather than license Gallup's Q12.

---

## Market Context

The employee engagement software market was valued at approximately $1.6 billion in 2024 and is growing at roughly 14% CAGR, with the virtual culture and connection sub-segment growing faster as hybrid and remote arrangements stabilise. Incumbent pricing spans $1–$3/user/month for lightweight Slack apps (HeyTaco, CultureBot, Doozy), $2–$5/user/month for mid-market matching tools (Donut, CoffeePals), and custom enterprise pricing for full suites (Cooleaf, RandomCoffee, Culture Amp). Primary buyers are HR and People Ops leaders, executives tracking retention risk, engineering managers seeking organic developer relationships, and DEI leaders measuring connection equity.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
