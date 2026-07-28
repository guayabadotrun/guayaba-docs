# Organizations

Every Guayaba account is organised into **organizations**. An organization owns its agents, GRAFTs, API keys, subscription, credits, and payment method — resources are never shared implicitly across organizations.

There are two kinds of organization:

- **Personal** — created automatically at signup. Every user has exactly one. It cannot be renamed to something impersonal, deleted, or invited into. In menus, breadcrumbs, and titles it is always shown as literal **Personal**.
- **Named organization** — created on demand from the dashboard or the API. Named organizations are how you collaborate with teammates: they support multiple members, roles, and invitations.

You can belong to any number of named organizations in addition to your Personal.

---

## Roles

Every membership carries exactly one role. Roles form a strict hierarchy: `owner` > `admin` > `member`. A higher role always includes everything a lower role can do.

| Role | Cardinality | What they can do |
|---|---|---|
| **Owner** | Exactly 1 per organization. Transferable. | Everything an admin can do, plus delete the organization, transfer ownership, change the subscription, purchase credits, and manage the payment method. |
| **Admin** | 0 or more | Rename the organization and change its avatar, mint master API keys, publish GRAFTs, invite people, promote/demote/remove other admins and members. |
| **Member** | 0 or more | Read the roster, chat with agents, use credits, create agents, mint agent-scoped API keys. Cannot invite anyone, publish GRAFTs, or mint master keys. |

Ownership is not shared. Transferring the owner role demotes the previous owner to **admin** in the same operation.

---

## Billing per organization

Billing is **per-organization**. Every organization — Personal or named — has:

- Its own Stripe customer and subscription (a named organization starts on the free **Hobby** tier automatically at creation).
- Its own credit balance and transaction ledger.
- Its own daily runtime allowance under Hobby limits.

Consequences:

- Credits and running-time budget bought or earned in one organization never leak into another.
- New named organizations do **not** receive welcome credits — those remain a one-time, one-per-user grant on your Personal organization.
- Two organizations cannot share a single Stripe customer. If you want teammates to pay for shared usage, create a named organization and add them as members.

See [Billing & Credits](../api-reference/billing.md) for the credit model and current plan pricing.

---

## API keys are organization-bound

Every master key (`g_master_…`) belongs to exactly one organization at the moment it is created. Any request made with that key acts on that organization — its agents, its billing, its GRAFTs. A master key can never be used to reach a different organization.

To operate against another organization, either:

- Switch to that organization in the dashboard header and mint a fresh master key from **Settings → API Keys**, or
- Call `POST /api/v1/tenants` from any master key to create a new organization; then mint a master key inside it — the key you used to create the organization does **not** gain access to it.

Agent-scoped keys (`g_agent_…`) inherit the organization of the agent they are attached to.

For the full auth model see [Authentication](../api-reference/authentication.md); for the organization management API see [Tenants API](../api-reference/tenants.md).

---

## Creating, switching, inviting, leaving

Everything can be done from the dashboard, and everything except account-level organization switching is also available on the Public API.

- **Create an organization** — **Settings → Organizations → New organization**, or `POST /api/v1/tenants`. Personal is not manageable and never appears in the list; it is only exposed as an entry in the switcher.
- **Switch organization** — open the profile-avatar dropdown at the bottom of the sidebar (Personal is always first, then organizations alphabetically). The dashboard reloads with the new organization active. A secondary switcher lives in the header of the Billing page so you can compare plans across organizations without navigating away.
- **Invite a teammate** — inside the organization: **Settings → Organizations → <Your organization> → Invitations → Invite**. Guayaba mints a one-time invite URL of the form `https://app.guayaba.run/invite/<token>` and shows it to you **once**, inside the confirmation dialog. The URL is auto-copied to your clipboard; Guayaba only stores `sha256(token)` server-side and cannot show the URL again. If you close the dialog before saving it, revoke the invitation and issue a new one. Send the URL to your teammate out-of-band; it works logged-out (they get a preview), expires after 7 days, and can only be used once.
- **Accept an invitation** — open the URL, log in (or sign up) as the invitee, and click **Accept**.
- **Leave an organization** — **Settings → Organizations → <Your organization> → General → Danger zone → Leave**. Owners must transfer ownership before they can leave.
- **Transfer ownership** — **Settings → Organizations → <Your organization> → General → Danger zone → Transfer ownership**. Choose an existing member. The former owner is demoted to admin.
- **Delete an organization** — **Settings → Organizations → <Your organization> → General → Danger zone → Delete**. Only the owner can delete, and only after every agent has been deleted, every API key revoked, the subscription cancelled, and no pending payments remain.

---

## Frequently asked questions

### Do welcome credits carry over into a new organization?

No. Named organizations are created with an active Hobby subscription but **no** welcome credits. The welcome credit grant is a one-time bonus on your Personal organization at signup.

### Can I move an existing agent to another organization?

Not today. Agents (and their sessions, files, GRAFT attachments, and API keys) stay in the organization where they were created. If you need the same setup elsewhere, recreate the agent in the target organization — using a GRAFT is the fastest way.

### What happens when I leave an organization?

Your membership row is removed and you can no longer see the organization's agents, credits, or billing. Master keys you minted while a member remain bound to that organization; they keep working until either:

- an owner or admin of that organization revokes them, or
- you leave via `POST /api/v1/tenant/leave`, which revokes the invoking key in the same transaction.

If you have been removed by someone else, ask them to revoke any keys you had issued from that organization.

### How do I recover a deleted organization?

You cannot. Deletion is a soft-delete for audit purposes: financial history is preserved server-side, but the organization, its agents, keys, and members are gone from every user-facing surface with no restore endpoint.

### Can two organizations share the same Stripe customer?

No. Every organization owns its own Stripe customer, subscription, and credit balance. Purchases, invoices, and payment methods do not cross organization boundaries.
