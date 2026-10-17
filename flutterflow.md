# FD53 — Flutter & FlutterFlow Integration

This document is the blueprint for wiring Flutter, FlutterFlow and other native
clients to FD53. It covers the credential model, the exact request shapes, and
the realtime strategy — plus the mistakes that break FlutterFlow integrations in
practice.

---

## 1. The credential model (read this first)

FD53 has **two** credentials, and conflating them is the single most common
source of integration bugs.

| Credential | Header | Authenticates | Safe in a client? |
|---|---|---|---|
| **API key** | `X-API-Key: fd53_live_…` | *Your backend* | **No** |
| **End-user token** | `Authorization: Bearer …` | *Your app's users* | **Yes** |

```
Flutter app / FlutterFlow app
        │
        │  ①  X-API-Key: fd53_live_…        (trusted backend only)
        ▼
   your server  ──── or ────  Cloud Function / Edge Function
        │
        │  ②  Authorization: Bearer <token>  (end user)
        ▼
      FD53
```

**A Flutter binary or a FlutterFlow web bundle is not a trusted backend.**
Anyone can unzip an APK or read the JS bundle and extract a shipped API key. If
your key has `*` scope, that is full read/write on your project.

Two correct options:

1. **Best** — a Cloud Function holding the API key proxies privileged calls.
2. **Acceptable** — ship only the end-user token, and give the key narrowly
   scoped permissions (`db:read`, `storage:read`) so a leaked key cannot write.

For an MVP built entirely in FlutterFlow, option 2 with a `db:read`-only key is
usually fine. Do not ship a `*` key to a client.

---

## 2. Configure FD53 in FlutterFlow

### Step 1 — create the project and key

In the FD53 dashboard: create a project, then an API key. Record two things
immediately — the key is shown once and never again.

### Step 2 — add the REST API group

In FlutterFlow: **Integrations → REST API → Add**, then configure.

| FlutterFlow field | Value |
|---|---|
| Group name | `FD53` |
| Base URL | `https://<deployment>.convex.site/v1` |
| Auth | `API Key` |
| Header name | `X-API-Key` |
| Header value | `fd53_live_…` |

### Step 3 — declare the calls

Add the endpoints you need. Each is one row in FlutterFlow's API group:

| Name | Method | Path |
|---|---|---|
| Register | POST | `/auth/register` |
| Login | POST | `/auth/login` |
| Me | GET | `/auth/me` |
| List Todos | GET | `/db/todos` |
| Insert Todo | POST | `/db/todos` |
| Update Todo | PATCH | `/db/todos/{rowId}` |
| Delete Todo | DELETE | `/db/todos/{rowId}` |
| Upload File | POST | `/storage/upload` |
| List Files | GET | `/storage/files` |
| Usage | GET | `/usage` |

**Or import the whole spec at once.** Point Swagger UI, Postman or any OpenAPI
importer at:

```
https://<deployment>.convex.site/v1/openapi.json
```

The spec is generated from the live route table, so it can never drift from
what the API actually serves. It declares the `X-API-Key` security scheme, so
most importers configure authentication for you.

---

## 3. End-user authentication flow

FlutterFlow cannot safely hold a secret, so sign-in happens from a user action.

### 3.1 Register

```http
POST /v1/auth/register
X-API-Key: fd53_live_...
Content-Type: application/json

{
  "email": "sam@example.com",
  "password": "correct-horse-battery",
  "name": "Sam Rivera"
}
```

```json
{
  "ok": true,
  "data": {
    "user": {
      "id": "kg2ab1c9x0…",
      "email": "sam@example.com",
      "name": "Sam Rivera",
      "role": "user",
      "permissions": ["db:read", "db:write", "storage:read", "storage:write"]
    },
    "profile": { "handle": "sam_r", "displayName": "Sam Rivera" },
    "token": "eyJhbGciOiJIUzI1NiIs…",
    "expiresIn": 86400,
    "sessionId": "9f2c1a7e4b6d8f03"
  }
}
```

> The token is a **flat field** on `data`, not nested inside a `session`
> object. Bind to `data.token`, not `data.session.token`.

`POST /v1/auth/login` returns the same shape.

### 3.2 Login

```http
POST /v1/auth/login
X-API-Key: fd53_live_...
Content-Type: application/json

{ "email": "sam@example.com", "password": "correct-horse-battery" }
```

### 3.3 Store the token

Persist `data.token` in FlutterFlow's **App State** so it survives page
reloads, then set it as the default header on every other call:

```
Authorization: Bearer {{FD53_TOKEN}}
```

In FlutterFlow, `App Settings → App State → add value → name FD53_TOKEN`,
then populate it from the login response with an "Update State" action.

### 3.4 Signed out

`GET /v1/auth/me` returns 401 when the token is invalid, expired or revoked.
Clear the stored token and route to the sign-in page. Handle this on **every**
authenticated call, not just `me` — sessions are revocable server-side and a
24-hour-old token stops working.

---

## 4. Table access — the part that surprises people

Each FD53 table has an `access` level that determines who can read and write it.

| Access | Behaviour |
|---|---|
| `public` | No token needed. Reads allowed, writes allowed. |
| `authenticated` | Needs any valid token. Reads and writes allowed. |
| `owner` | A user may only read and write rows **they created**. |

The distinction matters: with `owner` access, a FlutterFlow collection bound to
`GET /v1/db/{table}` automatically scopes to the current user, so you never
write per-user filter logic in the UI. Set a table to `owner` for notes,
comments, drafts and anything personal.

> **Scope note.** Table `access` governs **end-user bearer tokens only**. A
> server-side call presenting only `X-API-Key` has full project access — that is
> how your backend does migrations and admin reads.

### Filter and sort from the UI

Any query parameter matching a column name becomes a filter, so FlutterFlow's
filter and sort controls map straight through:

```
GET /v1/db/todos?status=open&limit=20&order=desc&orderBy=createdAt
```

Up to 10 filters per request. `limit` is capped at 200 rows per page.

---

## 5. Storage uploads

FD53 accepts three upload styles so FlutterFlow, curl and native clients all
work without special-casing.

### From FlutterFlow (multipart)

FlutterFlow's image picker sends `multipart/form-data`. Use the **File Upload**
action rather than a generic REST call.

```
POST /v1/storage/upload
X-API-Key: fd53_live_...
Content-Type: multipart/form-data; boundary=…

file: <binary>
bucket: avatars
```

### From raw Dart

```dart
import 'dart:io';

final request = http.MultipartRequest(
  'POST',
  Uri.parse('$baseUrl/v1/storage/upload'),
)
  ..headers['X-API-Key'] = apiKey
  ..headers['Authorization'] = 'Bearer $token'
  ..files.add(await http.MultipartFile.fromPath('file', file.path))
  ..fields['bucket'] = 'avatars';

final response = await request.send();
```

### Buckets and their fixed visibility

| Bucket | Visibility | Accepts |
|---|---|---|
| `avatars` | public | image, video, audio |
| `post-media` | public | image, video, audio |
| `receipts` | **private** | any |

Bucket visibility is **fixed by the bucket and enforced on upload**. Sending
`visibility: "public"` for a file in `receipts` is ignored — a private file
cannot be published through the API. This is deliberate: it makes "accidentally
exposed a customer document" structurally impossible.

### Limits

25 MB per file · 250 MB per project · 50 tables · 200 rows per page.
Quota and limit errors return 413 with a specific code.

---

## 6. Realtime

### The current model

FD53 persists events to a channel and clients read them back:

```http
POST /v1/realtime/publish
X-API-Key: fd53_live_...
Authorization: Bearer <token>

{ "channel": "room:dm:abc123", "type": "message", "payload": { "text": "hi" } }
```

```http
GET /v1/realtime/channels/room:dm:abc123/events
```

This is correct and durable, and it is the right choice for FlutterFlow, whose
runtime cannot hold a WebSocket open reliably — especially in background or on
mobile web.

### Poll sensibly

```dart
Timer? _timer;

void startListening(String channel) {
  _timer = Timer.periodic(const Duration(seconds: 3), (_) async {
    final events = await fd53.realtime.events(channel, since: _lastSeenAt);
    if (events.isNotEmpty) {
      _lastSeenAt = events.last.createdAt;
      setState(() => _messages.addAll(events));
    }
  });
}

@override
void dispose() {
  _timer?.cancel();   // always cancel — a leaked timer outlives the screen
  super.dispose();
}
```

Three rules that keep polling cheap and correct:

1. **Always pass `since`** — otherwise you re-download the whole history every
   tick.
2. **Always cancel the timer on dispose.**
3. **Back off when idle** — stop at 3s and stretch to 15–30s after several
   empty polls; reset to 3s the moment an event arrives.

### The WebSocket path (native Flutter)

For a native Flutter app needing true low latency, the future WebSocket hub
speaks:

```
wss://<deployment>/realtime?token=<bearer>
  → {"type":"subscribe","channel":"room:dm:abc123"}
  ← {"type":"event","channel":"room:dm:abc123","data":{…}}
  ← {"type":"presence","channel":"room:dm:abc123","online":3}
  ← {"type":"typing","channel":"room:dm:abc123","userId":"kg2…"}
```

While the hub is in the roadmap, polling through the REST API is the supported
path and covers chat, notifications and presence adequately. See
[`roadmap.md`](./roadmap.md).

---

## 7. Native Flutter SDK (Dart)

A minimal client, dependency-free apart from `http`:

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;

class Fd53 {
  Fd53({required this.baseUrl, required this.apiKey});

  final String baseUrl;
  final String apiKey;
  String? _token;

  String? get token => _token;
  set token(String? value) => _token = value;

  Map<String, String> get _headers => {
        'X-API-Key': apiKey,
        'Content-Type': 'application/json',
        if (_token != null) 'Authorization': 'Bearer $_token',
      };

  /// Every FD53 response uses the same envelope, so unwrap it in one place.
  dynamic _unwrap(http.Response res) {
    final body = jsonDecode(res.body);
    if (body['ok'] != true) {
      final err = body['error'] ?? const {};
      throw Fd53Exception(
        res.statusCode,
        err['code'] ?? 'unknown_error',
        err['message'] ?? 'Request failed',
      );
    }
    return body['data'];
  }

  Future<Map<String, dynamic>> login(String email, String password) async {
    final res = await http.post(
      Uri.parse('$baseUrl/auth/login'),
      headers: _headers,
      body: jsonEncode({'email': email, 'password': password}),
    );
    final data = _unwrap(res);
    token = (data['session'] as Map)['token'];
    return (data['user'] as Map).cast<String, dynamic>();
  }

  Future<List<dynamic>> rows(String table, {Map<String, String>? query}) async {
    final uri = Uri.parse('$baseUrl/db/$table').replace(
      queryParameters: (query ?? {}).map((k, v) => MapEntry(k, '$v')),
    );
    final res = await http.get(uri, headers: _headers);
    return (_unwrap(res) as Map)['items'] as List;
  }

  Future<Map<String, dynamic>> insert(String table, Map<String, dynamic> doc) async {
    final res = await http.post(
      Uri.parse('$baseUrl/db/$table'),
      headers: _headers,
      body: jsonEncode(doc),
    );
    return (_unwrap(res) as Map).cast<String, dynamic>();
  }

  Future<void> logout() async {
    await http.post(Uri.parse('$baseUrl/auth/logout'), headers: _headers);
    token = null;
  }
}

class Fd53Exception implements Exception {
  Fd53Exception(this.status, this.code, this.message);
  final int status;
  final String code;
  final String message;

  /// Branch on `code`, not on the message — messages change, codes do not.
  bool get isAuthFailure => status == 401 || status == 403;
  bool get isRateLimited => status == 429;

  @override
  String toString() => 'Fd53Exception($status $code): $message';
}
```

### Error handling

Every error shares one envelope, so handle it in exactly one place:

```json
{ "ok": false, "error": { "code": "rate_limited", "message": "…" } }
```

| Status | Code | What to do |
|---|---|---|
| 401 | `missing_api_key` | Server misconfiguration — check the key |
| 401 | `invalid_api_key` | Wrong or revoked key |
| 401 | `key_expired` | Rotate the key |
| 401 | `token_revoked` / `session_expired` | Clear the token, send the user to sign in |
| 403 | `insufficient_scope` | The key needs a broader scope |
| 403 | `ip_not_allowed` | Key restricted to a different network |
| 403 | `forbidden` | The user's role lacks the permission |
| 403 | `blocked_by_waf` | A URL rule tripped; inspect the path |
| 429 | `rate_limited` | Back off, honour `Retry-After` |
| 413 | `storage_quota_exceeded` | Delete files or upgrade the plan |
| 5xx | `internal_error` | Retry with backoff, and quote the request id |

Always honour `Retry-After` on a 429, and always log `X-FD53-Request-Id` — it
is the correlation id that lets FD53 find the exact request in its logs.

---

## 8. FlutterFlow-specific pitfalls

Real failures, and what causes them.

**1. Forgetting the `X-API-Key` header on preflight.**
FD53 answers `OPTIONS` with CORS headers, so preflight succeeds; the real call
then 401s if the key is missing from the group config. Configure the key at the
**group** level, not per-call.

**2. Treating the response as the payload.**
FD53 wraps everything in `{ "ok", "data" }`. A binding pointed at `email`
instead of `data.user.email` yields null. Either bind through `data`, or add a
transform that unwraps it.

**3. Hardcoding the bearer token.**
A token in a page parameter is visible in the editor and shared across builds.
Use App State, and clear it on sign-out.

**4. Ignoring 401 on a page that isn't `me`.**
Sessions are revocable server-side. Guard every authenticated call, not just
the profile fetch.

**5. Uploading without setting `bucket`.**
Omitting `bucket` infers one from the MIME type, which puts documents in
`receipts` (private). Set `bucket` explicitly so visibility is never a surprise.

**6. Retrying a 429 immediately.**
It extends the block. Honour `Retry-After`.

**7. Large `limit` values.**
Capped at 200 rows per page. Paginate rather than requesting a huge page.

---

## 9. Verification checklist

Before shipping an FD53-backed FlutterFlow project:

- [ ] The API key is stored in the API group config, not in a page parameter
- [ ] The key's scopes are the narrowest that work (`db:read` if read-only)
- [ ] The key has an expiry set
- [ ] The token lives in App State and is cleared on sign-out
- [ ] Tables holding per-user data use `access: owner`
- [ ] Uploads set `bucket` explicitly
- [ ] Every authenticated call handles 401 by signing the user out
- [ ] 429 responses honour `Retry-After`
- [ ] Polling passes `since` and cancels its timer
- [ ] `GET /v1/usage` shows consumption that looks plausible
- [ ] `X-FD53-Request-Id` is logged on errors

---

## 10. Related

- [`architecture.md`](./architecture.md) — platform design
- [`roadmap.md`](./roadmap.md) — WebSocket hubs, SDKs, remaining integrations
- Live API reference — `/dashboard/docs` in your deployment
- Machine-readable spec — `https://<deployment>/v1/openapi.json`