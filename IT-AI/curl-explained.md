# cURL — Explained Simply

## What is cURL?

**cURL** ("Client URL") is a command-line tool for sending HTTP(S) requests
to a server and getting the response back — no browser needed. It's how
you talk to APIs, download files, or test a web server directly from the
terminal.

Think of it like a browser with no visual interface: you type an address,
maybe some extra data, and cURL shows you exactly what comes back — headers,
body, status code.

It's installed by default on macOS, Linux, and modern Windows (`curl` works
in PowerShell and Git Bash out of the box).

---

## How it works

You run `curl` followed by a URL and some flags describing the request:

```bash
curl [options] <url>
```

Under the hood, cURL:
1. Opens a network connection to the server (HTTP or HTTPS).
2. Sends a request: a method (`GET`, `POST`, `PUT`, `DELETE`...), headers,
   and optionally a body.
3. Receives the response: a status code (`200`, `404`, `500`...), headers,
   and a body (often JSON, HTML, or a file).
4. Prints the body to your terminal (unless you save it to a file).

### Common flags

| Flag | Meaning | Example |
|---|---|---|
| `-X` | HTTP method | `-X POST` |
| `-H` | Add a header | `-H "Content-Type: application/json"` |
| `-d` | Send data (body) | `-d '{"name":"Matt"}'` |
| `-o` | Save output to a file | `-o result.json` |
| `-i` | Show response headers too | `-i` |
| `-s` | Silent mode (no progress bar) | `-s` |
| `-L` | Follow redirects | `-L` |

---

## Real use-case examples

### 1. Fetch a webpage's HTML
```bash
curl https://example.com
```
Prints the raw HTML to your terminal.

### 2. Call a JSON API (GET request)
```bash
curl https://api.github.com/users/octocat
```
Returns JSON info about the GitHub user `octocat`.

### 3. Send data to an API (POST request)
```bash
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Matt", "email": "matt@example.com"}'
```
Creates a new user by sending JSON in the request body.

### 4. Authenticate with an API key
```bash
curl https://api.openai.com/v1/models \
  -H "Authorization: Bearer $OPENAI_API_KEY"
```
`$OPENAI_API_KEY` here is an [environment variable](environment-variables-explained.md)
— the key never appears typed out in the command itself.

### 5. Download a file
```bash
curl -o photo.jpg https://example.com/photo.jpg
```

### 6. Check if a website is up (just the status code)
```bash
curl -I https://example.com
```
Returns only the response headers (including the `200 OK` or `404` status),
without downloading the whole page.

---

## Running the same request in Python

### Using `requests` (most common)
```python
import requests

response = requests.get(
    "https://api.github.com/users/octocat"
)
print(response.status_code)   # e.g. 200
print(response.json())        # parsed JSON body
```

POST with JSON body + auth header:
```python
import os
import requests

response = requests.post(
    "https://api.example.com/users",
    headers={
        "Content-Type": "application/json",
        "Authorization": f"Bearer {os.environ['OPENAI_API_KEY']}"
    },
    json={"name": "Matt", "email": "matt@example.com"}
)
print(response.status_code, response.json())
```

*(Install with `pip install requests` if not already available.)*

### Using only the standard library (no install needed)
```python
import urllib.request
import json

req = urllib.request.Request(
    "https://api.github.com/users/octocat",
    headers={"Accept": "application/json"}
)
with urllib.request.urlopen(req) as resp:
    data = json.loads(resp.read())
    print(data)
```

---

## Running the same request in JavaScript

### In Node.js (modern versions, built-in `fetch`)
```javascript
const response = await fetch("https://api.github.com/users/octocat");
const data = await response.json();
console.log(data);
```

POST with JSON body + auth header:
```javascript
const response = await fetch("https://api.example.com/users", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": `Bearer ${process.env.OPENAI_API_KEY}`
  },
  body: JSON.stringify({ name: "Matt", email: "matt@example.com" })
});

console.log(response.status);
console.log(await response.json());
```

### In the browser (also `fetch`, same API)
```javascript
fetch("https://api.github.com/users/octocat")
  .then(res => res.json())
  .then(data => console.log(data));
```

---

## Quick equivalence cheat sheet

| cURL | Python (`requests`) | JavaScript (`fetch`) |
|---|---|---|
| `curl <url>` | `requests.get(url)` | `fetch(url)` |
| `-X POST` | `requests.post(url)` | `method: "POST"` |
| `-H "K: V"` | `headers={"K": "V"}` | `headers: { K: "V" }` |
| `-d '{"a":1}'` | `json={"a": 1}` | `body: JSON.stringify({a:1})` |
| response body | `.json()` or `.text` | `await res.json()` |
| status code | `.status_code` | `.status` |

---

## Why this matters

Many APIs (Stripe, OpenAI, GitHub, internal company APIs) publish their
documentation as cURL examples, because it's the simplest, most universal
way to show a request — no programming language assumed. Once you can read
a cURL example, translating it to Python or JavaScript for your own code is
mostly a mechanical swap using the table above.
