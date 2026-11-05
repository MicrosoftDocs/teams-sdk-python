---
title: Migrate a Python Teams SDK Application to HTTPX2
description: Upgrade your Teams SDK for Python app to HTTPX2 in version 2.2. Learn how to update exception handling, HTTP clients, and certificate trust settings.
ms.date: 11/05/2026
ms.topic: reference
---

# Migrate a Python Teams SDK application to HTTPX2

Beginning with version 2.2, Teams SDK for Python uses [HTTPX2](https://pydantic.dev/docs/httpx2/get-started/) for HTTP client operations instead of [HTTPX](https://www.python-httpx.org/). Use this guide to help navigate HTTP client changes when upgrading an existing Teams SDK for Python application to use SDK version 2.2.

There are two important changes:

1. HTTP objects and exceptions exposed by the SDK now come from `httpx2`, not `httpx`.
2. HTTPS connections trust the operating system's certificate store by default, rather than certifi's certificate bundle.

If you only use high-level SDK features, such as sending Teams messages, your handlers generally do not need transport-related changes. You should still check HTTPS connectivity in the environment where your application runs.

## What do I need to update?

| Your application... | What to do |
| --- | --- |
| Catches `httpx` exceptions from Teams SDK calls | Change those catches to `httpx2`. |
| Uses HTTP request/response types, middleware, or interceptors | Update SDK-facing HTTP types and runtime checks. |
| Supplies an HTTP client or mock transport to the SDK | Replace it with the matching HTTPX2 client or transport. |
| Runs in a container or uses company-issued certificates | Check the certificate setup described below. |
| Uses other libraries that depend on HTTPX | Keep the HTTP library those libraries require. Both can coexist. |

## 1. Upgrade your dependencies together

Upgrade the Teams packages you use to a compatible release set, especially `microsoft-teams-apps`, `microsoft-teams-api`, and `microsoft-teams-common`. Avoid upgrading only the shared HTTP package underneath older apps/API packages.

Use your normal package manager to update the dependencies and lockfile. If your application imports `httpx2` directly, declare it as an application dependency too. The migration currently uses:

```toml
"httpx2>=2.13.1,<3"
```

Do not force-remove `httpx` or `certifi` from your environment. Other dependencies may still need them.

## 2. Update exception handling and HTTP types

Here is an existing error handler around the Teams SDK's common `Client`.

**Before:**

```python
import httpx

try:
    response: httpx.Response = await client.get(url)
except httpx.HTTPStatusError as error:
    handle_status(error.response.status_code)
```

**After:**

```python
import httpx2

try:
    response: httpx2.Response = await client.get(url)
except httpx2.HTTPStatusError as error:
    handle_status(error.response.status_code)
```

These snippets go inside your existing async function; `client`, `url`, and `handle_status` represent your application's existing code.

**An old `except httpx.HTTPStatusError` will not catch an SDK HTTPX2 error.** The classes are different, and `httpx2.HTTPStatusError` is not a subclass of the HTTPX version. If HTTPX remains installed, the old import still works, which can make this easy to miss.

Apply the same review to `RequestError`, `TimeoutException`, and other HTTPX exceptions that handle SDK calls. Keep your existing decisions about which errors to handle or propagate.

Also update SDK-facing uses of:

| Before | After |
| --- | --- |
| `httpx.Request` | `httpx2.Request` |
| `httpx.Response` | `httpx2.Response` |
| `httpx.AsyncClient` | `httpx2.AsyncClient` |
| `httpx.MockTransport` | `httpx2.MockTransport` |

Check type annotations, `isinstance` checks, middleware return values, interceptors, test fixtures, and mock patch targets.

**Do not run a blanket replacement across every integration.** For example, an external Graph or AI SDK may still expect HTTPX clients. Match the client and exceptions to the library receiving the call.

## 3. Replace custom clients and mocks

If an SDK API you use accepts a raw HTTP client, create an HTTPX2 client for that argument:

```python
# Before
raw_client = httpx.AsyncClient()

# After
raw_client = httpx2.AsyncClient()
```

This is a change to the actual object, not just its annotation. Keep your existing timeout, headers, and supported customization, and continue closing the client when its owning application component is finished with it. An async context manager is appropriate for a bounded operation.

For mocked requests, use matching HTTPX2 objects:

```python
import httpx2


def respond(request: httpx2.Request) -> httpx2.Response:
    return httpx2.Response(200, json={"ok": True})


transport = httpx2.MockTransport(respond)
```

Construct the test client with `httpx2.AsyncClient(transport=transport)` and pass it through an existing supported injection point. Do not modify private client attributes such as `_transport`.

## 4. Check HTTPS certificate trust

A certificate authority, or **CA**, is an authority your application trusts to identify HTTPS servers. A **CA bundle** is a file containing trusted CA certificates.

Previously, default destination-server verification used certifi's bundle. HTTPX2 instead uses the operating system's trust store through `truststore`. This can change which servers your application trusts, even when your Python code is unchanged.

For clients created by the SDK with their default settings:

| Environment configuration | Trust source |
| --- | --- |
| `SSL_CERT_FILE` contains a nonempty value | The specified CA bundle file |
| Otherwise, `SSL_CERT_DIR` contains a nonempty value | The specified OpenSSL-compatible CA directory |
| Neither is configured | The operating system's trust store |

**These variables replace the default trust source; they do not add to it.** If both are set, `SSL_CERT_FILE` wins. A custom bundle must contain all the authorities your application needs, including public roots if it connects to public services.

### If you use standard public HTTPS services

Check that the deployment has a usable system CA store. No custom CA configuration may be needed.

### If you use company certificates or a corporate proxy

Install the approved company CA into the deployment's system trust store, or configure an approved CA bundle before starting the application.

Linux/macOS shell:

```bash
export SSL_CERT_FILE=/path/to/approved-ca-bundle.pem
```

PowerShell:

```powershell
$env:SSL_CERT_FILE = "C:\certificates\approved-ca-bundle.pem"
```

Alternatively, use `SSL_CERT_DIR` with a correctly prepared OpenSSL CA directory; a plain folder of certificate files may not be sufficient.

Make sure the path exists and is readable by the application. Restart the application after changing these settings so its clients are recreated.

### If you deploy in a minimal or distroless container

The image may not contain the CA certificates HTTPX2 needs. Install the image's supported CA certificate package, or supply an approved bundle in the image or through a mount and configure its path.

Test from **inside the deployed image**. A container does not automatically inherit your development machine's certificate store. If you previously modified certifi's bundle, that customization is not automatically used by HTTPX2.

**Do not fix certificate errors by disabling verification.** The SDK does not automatically retry with another trust store or fall back to HTTPX.

### If you already configure raw clients yourself

An existing raw-client injection point can use an explicit context:

```python
import ssl

import httpx2

tls = ssl.create_default_context(cafile="/path/to/approved-ca-bundle.pem")
```

Pass it to `httpx2.AsyncClient(verify=tls)` and manage the client's lifetime as usual. This is a raw HTTPX2 client option, not a `ClientOptions` field. Custom raw-client settings are not automatically copied through App's client cloning; use the supported environment settings for SDK-created clients rather than assuming one injected client configures every request path.

An **HTTPS proxy** also has its own certificate check, separate from the destination server's check. For explicit HTTPX2 proxies, `httpx2.Proxy(..., ssl_context=...)` configures that separate context. Test your actual proxy route; setting the destination's `verify` context alone does not configure both.

## 5. Check logs and test before rollout

If you filter HTTP transport logs, update the SDK-related logger names from `httpx`/`httpcore` to `httpx2`/`httpcore2`. Keep the old filters too if other integrations still use HTTPX.

Use this short upgrade checklist:

- Confirm SDK exception handlers and custom clients use HTTPX2.
- Run the workflows your app depends on: messaging, authentication, files, and Socket Mode where applicable.
- Test HTTPS from the real deployment environment, including any corporate proxy or custom CA configuration.
- Check that failures still reach your error handlers and that streams and caller-owned clients are cleaned up correctly.
- Keep a known-good package set and lockfile available for rollback.

Mock transports can check request and error handling, but **they do not verify real TLS or service connectivity**.

### Common upgrade problems

| Symptom | Check first |
| --- | --- |
| An error escapes a handler that worked before | Does the handler still catch `httpx` instead of `httpx2`? |
| A custom client or test mock fails | Are the client, transport, responses, and exceptions from the expected library? |
| HTTPS works locally but not in a container | Does the container have the required CA certificates and readable configured paths? |
| Public services fail after configuring a private CA | Does the replacement bundle also contain the required public roots? |
| An HTTPS proxy fails | Is the proxy's own certificate trusted, as well as the destination's? |

If you need to roll back, restore your compatible Teams package set and lockfile together. Do not mix the old shared HTTP client with migrated SDK packages or disable TLS verification.

## See also

- [HTTPX2 migration documentation](https://pydantic.dev/docs/httpx2/get-started/migration/). Certificate-selection behavior described here is based on HTTPX2/HTTPcore2 v2.13.1.
