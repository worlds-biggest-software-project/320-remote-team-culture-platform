# Standards & API Reference

> Project: Remote Team Culture Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

No single ISO standard governs employee engagement or team culture software directly. The following ISO standards are applicable to the data handling, security, and HR process dimensions of this project:

- **ISO/IEC 27001:2022 — Information Security Management Systems**
  URL: https://www.iso.org/standard/27001
  The primary information security management standard. Relevant to storing and processing sensitive employee engagement data, participation signals, and mood/wellness inputs. Enterprise buyers will ask for ISO 27001 alignment or certification.

- **ISO/IEC 27701:2019 — Privacy Information Management**
  URL: https://www.iso.org/standard/71670.html
  Extension to ISO 27001 covering privacy information management, including roles of data controllers and processors. Directly applicable to GDPR and CCPA compliance obligations when processing employee personal data (wellness inputs, engagement scores, matching preferences).

- **ISO 30414:2018 — Human Resource Reporting**
  URL: https://www.iso.org/standard/69338.html
  Guidelines for internal and external human capital reporting. Relevant when exposing workforce engagement metrics and culture analytics to leadership or external stakeholders. Provides a standard taxonomy for people metrics (employee engagement, leadership trust, retention risk) that culture analytics modules can align to.

---

### W3C & IETF Standards

- **RFC 6749 — The OAuth 2.0 Authorization Framework**
  URL: https://datatracker.ietf.org/doc/html/rfc6749
  The authorisation framework used by Slack, Microsoft Teams, Google Workspace, and all major HRIS APIs to grant delegated access. Required for connecting the platform to third-party services on behalf of users. The Authorisation Code flow with PKCE (RFC 7636) is recommended for web apps.

- **RFC 7636 — Proof Key for Code Exchange (PKCE) for OAuth Public Clients**
  URL: https://datatracker.ietf.org/doc/html/rfc7636
  Security extension to the OAuth 2.0 Authorisation Code flow for public clients. Required when the platform acts as an OAuth client connecting to Slack or Teams from a JavaScript/mobile front-end where client secrets cannot be kept confidential.

- **RFC 7643 — SCIM Core Schema**
  URL: https://datatracker.ietf.org/doc/html/rfc7643
  Defines the data model for user and group objects in the System for Cross-domain Identity Management (SCIM). Relevant for importing and maintaining employee rosters from HR systems and identity providers (Okta, Azure AD, Workday).

- **RFC 7644 — SCIM Protocol**
  URL: https://datatracker.ietf.org/doc/html/rfc7644
  Defines the HTTP-based REST protocol for SCIM operations (create, read, update, delete users and groups). Implementing a SCIM server endpoint allows enterprise Identity Providers to automatically provision and deprovision employees from the platform without manual admin action.

- **RFC 7519 — JSON Web Token (JWT)**
  URL: https://datatracker.ietf.org/doc/html/rfc7519
  Standard for encoding claims in a compact, URL-safe token. Used for session authentication tokens and inter-service communication in the platform backend. Required when consuming the Slack Events API or Teams Bot Framework using signed payloads.

- **RFC 4791 — CalDAV: Calendaring Extensions to WebDAV**
  URL: https://datatracker.ietf.org/doc/html/rfc4791
  Protocol for calendar access and scheduling. Relevant for features that book virtual coffee meetings directly in participants' calendars. Google Calendar exposes a CalDAV interface; Outlook uses the Microsoft Graph API instead.

- **RFC 8288 — Web Linking**
  URL: https://datatracker.ietf.org/doc/html/rfc8288
  Standard for expressing typed links between resources in HTTP responses. Used for paginated API responses (next, prev, first, last link relations) in the platform's REST API.

---

### Data Model & API Specifications

- **OpenAPI Specification 3.1 (OAS 3.1)**
  URL: https://spec.openapis.org/oas/v3.1.0
  The standard for documenting RESTful HTTP APIs. The platform's public API and all internal service interfaces should be described using OAS 3.1 YAML/JSON documents to enable code generation, automated testing, and third-party integration development.

- **JSON Schema Draft 2020-12**
  URL: https://json-schema.org/specification.html
  Used within OpenAPI 3.1 for request/response body validation. Relevant for defining the schemas of engagement event payloads, survey response bodies, and matching configuration objects.

- **Slack Block Kit**
  URL: https://api.slack.com/block-kit
  Slack's UI framework for structuring interactive messages. Required for all engagement rituals, match notifications, recognition shoutouts, and celebration messages delivered inside Slack. Supports interactive elements (buttons, dropdowns, modals) enabling in-Slack RSVP and feedback flows.

- **Microsoft Adaptive Cards**
  URL: https://adaptivecards.io/
  Microsoft's cross-platform interactive card format, used natively in Microsoft Teams. Required for Teams-native delivery of match notifications, pulse survey questions, and recognition messages. Teams supports Adaptive Card v1.5 for bot-sent cards.

- **Slack Events API**
  URL: https://api.slack.com/events-api
  Webhook-based event subscription model for receiving Slack workspace events (messages, reactions, user joins). Used to listen for recognition triggers, respond to slash commands, and ingest participation signals without polling.

---

### Security & Authentication Standards

- **OAuth 2.0 with PKCE (RFC 6749 + RFC 7636)**
  See IETF section above. All third-party OAuth connections (Slack, Teams, Google, HRIS) must use the Authorisation Code + PKCE flow. Implicit flow is deprecated and must not be used.

- **OpenID Connect 1.0 (OIDC)**
  URL: https://openid.net/connect/
  Identity layer on top of OAuth 2.0. Used for authenticating admin users via SSO providers (Google Workspace, Okta, Azure AD). Required for enterprise Single Sign-On support.

- **SAML 2.0**
  URL: https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf
  XML-based SSO standard widely used in enterprise HR environments. Required alongside OIDC to support customers whose identity providers (Okta, Ping, ADFS) prefer SAML over OIDC.

- **OWASP Top 10 (2021)**
  URL: https://owasp.org/www-project-top-ten/
  Reference checklist for the most critical web application security risks. Relevant throughout development: injection, broken access control, and insecure direct object references are especially pertinent for a multi-tenant platform handling employee personal data.

- **GDPR (EU 2016/679)**
  URL: https://gdpr-info.eu/
  The primary data protection regulation for EU/EEA users. Imposes obligations on lawful basis for processing employee engagement data, data minimisation, consent management, right to erasure, and Data Protection Impact Assessments (DPIAs) for wellness-tracking features. Processing engagement signals without a valid lawful basis (legitimate interest or explicit consent) exposes the platform to regulatory risk.

- **CCPA / CPRA (California)**
  URL: https://oag.ca.gov/privacy/ccpa
  California consumer privacy law with obligations similar to GDPR for California-resident employees. Requires disclosure, opt-out rights for data sale, and data subject access request handling.

---

### MCP Server Specifications

The Model Context Protocol (MCP) is relevant if the platform exposes engagement analytics to AI assistants or agent-based workflows.

- **Model Context Protocol (MCP)**
  URL: https://modelcontextprotocol.io/
  Open protocol for connecting AI models to external data sources and tools. An MCP server exposing engagement metrics, network graph data, and pulse-survey results would allow AI assistants (Claude, Copilot) to retrieve culture insights on demand for leadership briefings, attrition-risk analysis, or automated culture narrative generation. Relevant for the AI-native differentiation described in the project's research.md.

---

## Similar Products — Developer Documentation & APIs

### Slack API

- **Description:** The primary integration platform for Slack-native engagement bots. Provides the Events API (webhooks), Web API (REST), Block Kit (interactive UI), and SCIM API for workspace user management.
- **API Documentation:** https://docs.slack.dev/
- **SDKs/Libraries:** Node.js SDK: https://github.com/slackapi/node-slack-sdk · Python SDK: https://github.com/slackapi/python-slack-sdk · Bolt (app framework): https://slack.dev/bolt-js/
- **Developer Guide:** https://docs.slack.dev/quickstart
- **Standards:** REST/JSON, Block Kit (proprietary UI format), SCIM 2.0 (RFC 7643/7644), OAuth 2.0 (RFC 6749)
- **Authentication:** OAuth 2.0 Authorisation Code flow; workspace bot tokens and user tokens

---

### Microsoft Teams API (Microsoft Graph + Bot Framework)

- **Description:** The integration surface for Microsoft Teams bots and apps. The Bot Framework handles conversational interactions; Microsoft Graph provides access to user data, calendar events, and Teams channel messages. Adaptive Cards deliver rich interactive UI.
- **API Documentation:** https://learn.microsoft.com/en-us/microsoftteams/platform/
- **SDKs/Libraries:** Bot Framework SDK (C#, Node.js, Python): https://github.com/microsoft/botframework-sdk · Microsoft Graph SDK: https://learn.microsoft.com/en-us/graph/sdks/sdks-overview
- **Developer Guide:** https://learn.microsoft.com/en-us/microsoftteams/platform/get-started/get-started-overview
- **Standards:** REST/JSON, Adaptive Cards v1.5, OAuth 2.0, OpenID Connect
- **Authentication:** OAuth 2.0 via Azure Active Directory (Entra ID); bot tokens issued by the Bot Framework Service

---

### Google Calendar API

- **Description:** RESTful API for creating, reading, and updating calendar events. Used to schedule virtual coffee meetings in participants' Google Calendars. Also exposes a CalDAV interface for standards-compliant clients.
- **API Documentation:** https://developers.google.com/workspace/calendar/api/guides/overview
- **SDKs/Libraries:** Google API Client Library (Python, Node.js, Java, Go): https://developers.google.com/api-client-library · CalDAV guide: https://developers.google.com/workspace/calendar/caldav/v2/guide
- **Developer Guide:** https://developers.google.com/workspace/calendar/api/quickstart/nodejs
- **Standards:** REST/JSON, CalDAV (RFC 4791), OAuth 2.0
- **Authentication:** OAuth 2.0 with Google Identity; CalDAV interface also requires OAuth 2.0 over HTTPS (basic auth deprecated)

---

### BambooHR API

- **Description:** REST API for a widely used mid-market HRIS. Provides access to employee records, directory data, time-off balances, and custom fields. Used to sync employee rosters (names, departments, tenure, birthdays) into the platform.
- **API Documentation:** https://documentation.bamboohr.com/reference/
- **SDKs/Libraries:** No official SDK; third-party wrappers available. Unified HRIS clients (Merge, Apideck, Unified.to) provide normalised access.
- **Developer Guide:** https://documentation.bamboohr.com/docs/getting-started
- **Standards:** REST/JSON; Changed Employees endpoint for incremental sync
- **Authentication:** HTTP Basic Auth with API key (per subdomain); no OAuth 2.0 on the v1 API

---

### Workday API

- **Description:** Enterprise HRIS with both SOAP (legacy) and REST APIs. Used by large enterprises for employee record management, organisational hierarchy, and custom reporting. REST API covers a growing subset of domains; complex operations still require SOAP/XML.
- **API Documentation:** https://community.workday.com/sites/default/files/file-hosting/restapi/ · SOAP: https://community.workday.com/node/204189
- **SDKs/Libraries:** No official SDK; Workday recommends using its REST Directory directly. Integration platforms (Boomi, MuleSoft, Apideck) abstract the Workday API.
- **Developer Guide:** https://www.apideck.com/blog/create-a-workday-rest-api-integration
- **Standards:** REST/JSON (modern endpoints), SOAP/XML (legacy endpoints), OAuth 2.0
- **Authentication:** OAuth 2.0 with Integration System User (ISU) credentials or API Client credentials registered in Workday tenant

---

### Together Platform API

- **Description:** Mentoring and matching platform with a public User REST API. Supports programmatic user management (create, update, list users) for custom HRIS integrations.
- **API Documentation:** https://www.togetherplatform.com/integrations (integration overview; full API docs behind customer login)
- **SDKs/Libraries:** No public SDK; REST endpoints only
- **Developer Guide:** Available to customers via the Together Help Centre
- **Standards:** REST/JSON
- **Authentication:** API key (Bearer token)

---

### Cooleaf API

- **Description:** Employee engagement and recognition platform exposing a REST API and webhooks. Supports authentication, read/write access to engagement data, and integration with Slack, Teams, Zapier, and Workday.
- **API Documentation:** https://apitracker.io/a/cooleaf (tracker page; official docs behind customer login at Cooleaf developer portal)
- **SDKs/Libraries:** No public SDK documented
- **Developer Guide:** Available to enterprise customers
- **Standards:** REST/JSON, OpenAPI/Swagger specifications available
- **Authentication:** OAuth 2.0

---

### Culture Amp API

- **Description:** Outbound (read-only) REST API for retrieving employee engagement survey results, participation data, and DEI analytics. Designed for integration with internal analytics systems and custom dashboards.
- **API Documentation:** https://docs.api.cultureamp.com/
- **SDKs/Libraries:** No official SDK; REST with JSON responses
- **Developer Guide:** https://docs.api.cultureamp.com/docs/customer-guide-for-customers
- **Standards:** REST/JSON; pagination follows standard Link header conventions (RFC 8288)
- **Authentication:** OAuth 2.0 (Client ID / Client Secret issued from the Culture Amp admin console)

---

## Notes

- **HRIS fragmentation**: The HRIS landscape is highly fragmented across BambooHR, Workday, Rippling, Personio, HiBob, UKG, and others. Rather than building and maintaining separate connectors for each, the platform should evaluate a unified HRIS middleware layer (Merge, Apideck, or Unified.to) that normalises employee data across providers into a single API.

- **Slack vs Teams feature parity**: Slack's Block Kit and Teams' Adaptive Cards are similar in concept but differ in capability and rendering; maintaining parity across both requires separate template implementations for interactive UI elements.

- **Emerging standard — MCP**: The Model Context Protocol is still in early adoption. An MCP server interface for exposing engagement analytics is a forward-looking investment that positions the platform as an AI-native data source for enterprise AI assistants, but production MCP tooling is not yet mature.

- **Gallup Q12 licensing**: The Q12 survey questions are proprietary to Gallup, Inc. Any engagement survey feature must use independently designed questions or other validated frameworks (e.g., Utrecht Work Engagement Scale, Kahn's Engagement Scale) to avoid licensing obligations.
