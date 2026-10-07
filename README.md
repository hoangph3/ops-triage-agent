# Ops Triage Agent

**AI Demo Challenge · Option 3 — Operations Automation Agent**

A Telegram assistant for an electronics shop (laptops, PCs, RAM, CPUs). Staff talk to it in Vietnamese. It triages
support tickets, looks up orders and subscriptions, **proposes** a next action (refund, cancel a plan, notify Slack,
draft a reply) and runs everyday shop operations (browse the catalog, place orders, add products, change stock and
price, delete orders).

The idea: **the model proposes, code disposes, and only a human approves.**

This repository holds the idea and the architecture only — no source code.

## Try it

**[@hoangph3_ops_bot](https://t.me/hoangph3_ops_bot)** on Telegram.

- Open to everyone: just message the bot, no sign-up.
- Mock data. Every side effect (refund, e-mail, Slack, deletion) is simulated, never sent.

| Say | What happens |
|---|---|
| `Cho tôi xem danh mục sản phẩm` | interactive catalog with buttons |
| `Đặt cho số 0377 777 777 tên Minh Anh 2 thanh RAM Corsair Vengeance DDR4 32GB` | a new customer is just a phone number |
| `Đặt cho số 0912345602 hai laptop Dell XPS 13 bản 32GB 1TB` | over the limit, so it waits in the approval queue |
| `Xử lý ticket của số 0901234501 bị trừ hai lần rồi cho tôi xem hàng chờ duyệt` | refund proposal with **Duyệt / Từ chối** buttons |
| `Xử lý ticket của số 0945678905` | a ticket trying to hijack the agent: escalated only |

## Architecture

```
 Telegram ─────► Channel layer ─────────► Agent runtime ────────────► Tool server
 (chat, buttons)  Telegram I/O, auth,      one LLM tool loop per      business tools, policy engine,
                  rendering                turn, streams progress     approval queue, JSON state, views
                       ▲                                                      │
                       └── "open a view" tool call → buttons rendered; button press → decision applied
```

- **Channel layer** — everything Telegram; no business logic, never talks to the model.
- **Agent runtime** — one persona with its own prompt (Vietnamese and English) and an explicit tool allow-list, called
  directly with a single tool loop per turn (about 6–18 s).
- **Tool server** — 19 tools over MCP (tickets, orders, subscriptions, catalog, order and product changes, "open this
  view" signals). It owns the policy engine, the approval queue and the data.

A turn: the agent reads (ticket, order, catalog) → **proposes** one action → the policy engine answers *automatic*,
*needs approval* or *blocked* → a person presses a button → the decision is re-checked against current data, applied
once and logged.

## Key design decisions

- **The model proposes, code disposes.** The agent executes nothing. A refund can never exceed what the customer paid,
  must be on that customer's own order, and always needs a human. The prompt asks the model to behave; the code
  guarantees it.
- **No "approve" tool.** Approval only exists as a human button press, checked again at that moment; pressing twice does nothing.
- **Customers are phone numbers** — no customer table, no e-mail. `0901234567`, `090 123 4567` and `+84901234567` are
  the same person. If one phone has several tickets, the agent asks which; it never guesses.
- **Ticket text is untrusted.** A suspected prompt injection can only produce an escalation.
- **No silent substitution.** If the requested variant is out of stock, the agent says so and asks.
- **Flexible, not obstructive.** Generous limits so a demo can explore many cases; the hard rules are: never refund more
  than was paid, never delete customer data, stock never goes negative.
- **Plain JSON, always fresh.** State is re-read on every call and written atomically; seed data reloads when its file
  changes. All money is VND (`27.990.000₫`).

| Action | Rule |
|---|---|
| draft reply, internal ticket, Slack message | automatic (low confidence → human) |
| refund, cancel subscription | human approval; refund ≤ paid and on the ticket's own order, else blocked |
| delete customer data (GDPR) | blocked, escalated to people |
| any action while injection is suspected | escalation only |
| create order | immediate; above 25.000.000₫ → human |
| delete order / product | always human approval |

## Known limitations

- Tested against a simulated Telegram transport, so issues only the real API shows (message length, formatting, rate
  limits) may remain.
- Mock data and simulated actions only — no real payments, shipping, e-mail or Slack.
- State is one JSON file with an in-process lock; concurrent writers could race.
- By default anyone the bot lets in can press the approval buttons; the audit log only records their Telegram id.

## For production

An evaluation set for triage quality; a real database with idempotency keys; real integrations behind the same action
interface; role-based approvers with a signed audit trail; a retry queue; PII redaction in logs.
