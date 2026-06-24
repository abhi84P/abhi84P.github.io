+++
title = "Building a Schema-Driven QR Generator in Blazor with Clean Architecture"
date = "2026-06-24"
weight = 1
description = "A .NET 10 Blazor Server app that generates and decodes QR codes, built on Clean Architecture with schema-driven payload types, server-side PDF/Word/email export, and Docker deployment to Render."
tags = ["dotnet", "blazor", "clean-architecture", "qrcode", "backend"]
+++

In this post we walk through a small but complete .NET 10 Blazor app that generates QR codes from typed forms and decodes uploaded QR images back into readable fields. The interesting part is not the QR encoding itself — it is how we drive the form entirely from each type's schema, and how we put every external concern behind an interface so the core logic never touches Blazor.

**Live demo:** [qr-generator-70oc.onrender.com](https://qr-generator-70oc.onrender.com/)

> Free-tier note: the instance spins down after ~15 minutes idle. First request cold-starts in roughly 30–50 seconds, then runs normally.

---

## What it does

Two features.

**Generate.** Pick a QR type from a dropdown, fill a form, download a PNG. The form fields change with the selected type because they are rendered from each type's schema — not hardcoded per page. Supported payloads:

| Type | Payload format |
|------|----------------|
| Bank Account | Human-readable labeled text |
| Employee Identity | vCard 3.0 (scanner-compatible contact) |
| WhatsApp | `https://wa.me/<phone>?text=<message>` |
| eSewa | JSON `{"eSewa_id":"...","name":"..."}` |
| Bank Account (QR) | JSON `{"accountNumber":...,"bankCode":...}` |

**Decode.** Upload a QR image (PNG/JPG, max 5 MB). The app decodes it, recognizes the payload format, and shows the fields in a readable layout. From there you can export to PDF, export to Word (.docx), or email the decoded info.

---

## Architecture

Clean Architecture with strict inward dependency flow. Each layer is a separate project so the core logic stays independent of the UI and third-party libraries.

```
Web  ──▶  Infrastructure  ──▶  Application  ──▶  Domain
```

| Project | Responsibility |
|---------|----------------|
| `Domain` | Enums, value objects (`QrType`, `DecodedQr`). Pure C#, no dependencies. |
| `Application` | Interfaces, payload builders, registry. Defines *what* the app does. |
| `Infrastructure` | Concrete implementations using third-party libraries. Defines *how*. |
| `Web` | Blazor UI. References Infrastructure only for DI wiring. |

The QR logic — encode, decode, export, mail — has no knowledge of Blazor. The UI could be swapped for a console app, a Web API, or MAUI without touching the core. Every external concern sits behind an interface, so the logic is unit-testable without a real SMTP server or file system.

---

## The key idea: schema-driven QR types

Each QR type implements one interface:

```csharp
public interface IQrPayloadBuilder
{
    QrType Type { get; }
    string DisplayName { get; }
    IReadOnlyList<QrField> Schema { get; }   // drives the dynamic form
    string Build(IReadOnlyDictionary<string, string?> values);
}
```

- `Schema` describes the fields — name, label, input kind, required, options.
- The Blazor page renders the form **from** the schema and collects values into a dictionary.
- `Build` turns those values into the payload string.

The JSON types (eSewa, Bank Account QR) share a `JsonPayloadBuilder` base that serializes whatever fields the schema declares — the JSON keys come from the schema, nothing is hardcoded in `Build`.

A `QrPayloadBuilderRegistry` collects all builders via dependency injection and feeds the dropdown.

**Adding a new type requires zero UI changes:**

1. Add a value to the `QrType` enum.
2. Add an `IQrPayloadBuilder` implementation.
3. Register it in `AddQrApplication`.

The dropdown and form pick it up automatically. That is the whole payoff of the schema approach — the form is a projection of data, not a hand-written page.

---

## Tech stack and rationale

| Concern | Library | Why |
|---------|---------|-----|
| QR encoding | **QRCoder** | Pure managed C#, zero transitive deps, MIT. Outputs PNG bytes directly — no `System.Drawing`, no Windows lock-in. |
| QR decoding | **ZXing.Net** | Standard .NET barcode library. Fed an `RGBLuminanceSource` so it stays decoupled from any image-library version. |
| Image loading | **SixLabors.ImageSharp 3.1.x** | Cross-platform decode. Pinned to 3.1.x — 2.x has a DoS advisory (GHSA-rxmq-m78w-7wmc) fixed only in 3.x. |
| PDF export | **QuestPDF** | Fluent API, minimal deps. |
| Word export | **DocumentFormat.OpenXml** | Generates .docx with no Office install. |
| Email | **MailKit** | Modern async SMTP client. |
| UI | **Blazor Web App (Interactive Server)** | Decode/export/mail all run server-side, no separate API. |

Because the UI is **Interactive Server**, decode/export/mail run on the server with no separate API — but it also means the app needs a live .NET host with WebSocket (SignalR) support. It **cannot** run on static hosts like GitHub Pages or Netlify static.

---

## Deployment: Docker on Render

The repo ships a `Dockerfile`, `.dockerignore`, and `render.yaml`. The Dockerfile is a two-stage build — the SDK image compiles and publishes, then a slim ASP.NET runtime image runs the result. The entrypoint binds to Render's injected `$PORT`:

```dockerfile
ENTRYPOINT ["sh", "-c", "ASPNETCORE_URLS=http://0.0.0.0:${PORT:-8080} dotnet QrGenerator.Web.dll"]
```

On Render: **New → Web Service**, connect the repo, it auto-detects the Dockerfile, pick the Free instance, deploy. First build ~3–5 minutes. GitHub pushes auto-deploy after that.

Email stays off until SMTP is configured — the button shows a graceful error instead of crashing. In a hosted environment, the `Smtp` settings come from environment variables with double-underscore nesting: `Smtp__Host`, `Smtp__Username`, `Smtp__Password`, `Smtp__FromAddress`.

---

## Known limitations

- **PII in QR codes** — payloads are plaintext. The Employee Identity type embeds blood group and address; anyone who scans it reads everything. A UI warning is worth adding.
- **Validation** — only *required* and basic format (phone digit count) are checked today. Email-format and account-number rules aren't enforced yet.
- **No live camera scan** — decode is upload-only. `IQrDecoder` is source-agnostic, so a camera source can be added later without touching the core.

---

The takeaway: a tiny app, but a clean seam between *what* it does and *how* it does it. Schema-driven forms and interface-bound infrastructure mean a new QR type is three lines and a new export format is one class. Try it: [qr-generator-70oc.onrender.com](https://qr-generator-70oc.onrender.com/).
