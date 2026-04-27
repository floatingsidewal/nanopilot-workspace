---
name: dns-management
description: >
  Create, update, list, and delete DNS records on Technitium DNS Server via its
  HTTP API. Use this skill when the user asks to add a DNS record, update a
  hostname, check what DNS entries exist, remove a stale record, or
  troubleshoot DNS resolution on the home network.
---

# DNS Management (Technitium)

Manage DNS records on the home network's Technitium DNS Server.

## Connection details

- **API endpoint**: `http://dns1.home.lan:5380/api`
- **Authentication**: API token resolved via the `secrets` skill. Look for
  Keychain entry `technitium-api-token` or environment variable
  `TECHNITIUM_API_TOKEN`.
- **Primary zone**: Determined by the network's search domain (e.g., `home.lan`).

## Authentication style — header, not query string

Technitium accepts the API token via either an `Authorization` header or a
`?token=` query parameter. **Always use the header form.** Query parameters
land in HTTP access logs and proxy logs, where the token leaks; headers do
not. Set the token as a shell variable once at the start of a session:

```bash
TOKEN=$(security find-generic-password -s technitium-api-token -w 2>/dev/null \
        || printenv TECHNITIUM_API_TOKEN)
test -n "$TOKEN" || { echo "Run secrets skill first"; exit 1; }
```

Every `curl` call below includes `-H "Authorization: Bearer ${TOKEN}"`. Never
substitute `?token=${TOKEN}`.

## Discovery

### List zones

```bash
curl -s -H "Authorization: Bearer ${TOKEN}" \
  "http://dns1.home.lan:5380/api/zones/list" | jq '.response.zones[].name'
```

### List records in a zone

```bash
curl -s -H "Authorization: Bearer ${TOKEN}" \
  "http://dns1.home.lan:5380/api/zones/records/get?domain=<zone>&zone=<zone>" \
  | jq '.response.records[]'
```

### Check if a record already exists

Before adding, query for it first:

```bash
curl -s -H "Authorization: Bearer ${TOKEN}" \
  "http://dns1.home.lan:5380/api/zones/records/get?domain=<fqdn>&zone=<zone>" \
  | jq '.response.records[]'
```

## Operations

### Add an A record

```bash
curl -s -H "Authorization: Bearer ${TOKEN}" \
  "http://dns1.home.lan:5380/api/zones/records/add?domain=<fqdn>&zone=<zone>&type=A&ipAddress=<ip>&ttl=3600"
```

### Add a CNAME record

```bash
curl -s -H "Authorization: Bearer ${TOKEN}" \
  "http://dns1.home.lan:5380/api/zones/records/add?domain=<alias>&zone=<zone>&type=CNAME&cname=<target>&ttl=3600"
```

### Update an existing record

Technitium uses delete + add (no in-place update):

```bash
curl -s -H "Authorization: Bearer ${TOKEN}" \
  "http://dns1.home.lan:5380/api/zones/records/delete?domain=<fqdn>&zone=<zone>&type=A&ipAddress=<old_ip>"

curl -s -H "Authorization: Bearer ${TOKEN}" \
  "http://dns1.home.lan:5380/api/zones/records/add?domain=<fqdn>&zone=<zone>&type=A&ipAddress=<new_ip>&ttl=3600"
```

### Delete a record

**This is a destructive operation. Ask the user for confirmation before
proceeding.** State which record will be removed and what services may be
affected.

```bash
curl -s -H "Authorization: Bearer ${TOKEN}" \
  "http://dns1.home.lan:5380/api/zones/records/delete?domain=<fqdn>&zone=<zone>&type=A&ipAddress=<ip>"
```

## Verification

After any change, verify resolution from the DNS server:

```bash
dig +short <fqdn> @10.1.1.10
```

If the cluster has a secondary DNS, verify both:

```bash
dig +short <fqdn> @10.1.1.10
dig +short <fqdn> @10.1.1.11
```

Technitium handles zone replication between primary and secondary
automatically. If the secondary doesn't resolve, check zone transfer settings.

## Safety rules

- **Always check if a record exists before adding** to avoid duplicates.
- **Never delete or modify records without asking the user first.** DNS changes
  can break service connectivity across the entire network. State which record
  will change, what its current value is, and what the new value will be.
- **Never bulk-delete records.** Process deletions one at a time with
  confirmation.
- **After every change, verify with `dig`** to confirm the record is live.
- Prefer A records over CNAMEs for infrastructure services.
- **Never log or echo `${TOKEN}`** in user-visible output. Use it only as a
  shell variable inside the curl invocation.
