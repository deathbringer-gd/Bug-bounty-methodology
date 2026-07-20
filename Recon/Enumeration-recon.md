# Wide subdomain enumeration

### Key concepts

The goal of wide subdomain enumeration is to discover as much authorised attack surface as possible, especially hidden development, staging, API, and administrative subdomains. Keep raw results, but use one canonical file for each type of result so that later stages do not miss earlier discoveries.

---

### Before starting

1. Read the program rules and create the target directory structure described in the main `README.md`.
    - Record in-scope domains, excluded domains, allowed testing types, rate limits, and the required contact or user-agent header in `scope.md`, `out-of-scope.md`, and `recon/automatic-recon/notes.md`.
    - Run the commands below from `target.com/recon/automatic-recon/` and replace `target.com` with the authorised root domain.
    - Do not scan third-party hosts, cloud storage, or non-production systems unless they are explicitly in scope.

### Steps for enumerating subdomains and their attack surface

1. Run `subfinder` to find subdomains from passive sources.
    Commands:
    - `subfinder -d target.com -all -silent | sort -u > domains-subfinder.txt`

2. Run `dnsx` to brute-force additional subdomains, then merge all domain sources into the canonical `domains.txt` file.
    Commands:
    - `dnsx -silent -d target.com -w "/home/g/Hacking/hacking-map/Bug bounty methodology/Recon/wordlists/namelist.txt" > domains-dnsx.txt`
    - `cat domains-subfinder.txt domains-dnsx.txt domains-manual.txt 2>/dev/null | sed 's/\.$//' | sort -u > domains.txt.tmp && mv domains.txt.tmp domains.txt`
    - Add reviewed results from certificate transparency, search engines, or other manual sources to `domains-manual.txt` before merging them.

3. Enumerate virtual hosts.
    Determine the default response size or word count first, then filter that baseline to reduce wildcard and default-host false positives.
    Commands:
    - `ffuf -w "/home/g/Hacking/hacking-map/Bug bounty methodology/Recon/wordlists/subdomains-top1million-110000.txt" -u "https://target.com/" -H "Host: FUZZ.target.com" -ac -rate 20 -of json -o ffuf-vhosts.json`
    - Review `ffuf-vhosts.json`, validate candidate hostnames with DNS or `httpx`, then add confirmed names to `domains-manual.txt` and merge them into `domains.txt`.

4. Use search engines and public sources to find relevant company information. Save the query, date, source URL, and relevant result in `dorks.txt`; do not treat search snippets as confirmed findings.
    Tools:
    - https://pentest-tools.com/information-gathering/google-hacking for example search operators.
    - https://ahrefs.com/blog/google-advanced-search-operators/ for additional search operators.

5. Resolve and probe the complete domain list to identify live web services.
    Commands:
    - `httpx -l domains.txt -silent -sc -ip -tech-detect -follow-redirects > live-with-info.txt`
    - `httpx -l domains.txt -silent -follow-redirects | sort -u > live.txt`
    - `sed -E 's#https?://([^/:]+).*#\1#' live.txt | sort -u > live-hosts.txt`
    - Keep `domains.txt` as all discovered in-scope names and `live.txt` as the canonical list of reachable web URLs.

6. Build an automatic technology inventory for every live web service.
    This creates a broad view of delivery infrastructure, server software, frameworks, client-side libraries, API protocols, identity providers, storage providers, and security controls. Treat automated fingerprints as leads and verify important ones during manual review.
    Commands:
    - `mkdir -p technology`
    - `httpx -l live.txt -silent -json -sc -title -tech-detect -web-server -ip -cdn -asn -follow-redirects > technology/httpx.jsonl`
    - `jq -r '[.url, ((.status_code // "") | tostring), (.title // ""), (.webserver // ""), ((.tech // []) | join(",")), (.cdn_name // ""), ((.asn // "") | tostring)] | @tsv' technology/httpx.jsonl > technology/technology-inventory.tsv`
    - `whatweb --input-file live.txt --no-errors -a 1 > technology/whatweb.txt` use as a second, low-impact fingerprinting source if WhatWeb is installed.
    - `grep -RniE 'graphql|graphiql|swagger|openapi|/api/|socket\.io|wss://|oauth|openid|saml|stripe|paypal|s3|firebase|cloudfront|vercel|netlify' technology/ > technology/technology-leads.txt`
    - Review `technology/technology-inventory.tsv` and `technology/technology-leads.txt`; record promising hosts and the evidence in `notes.md` before moving them to manual recon.

7. Crawl live web applications and merge all discovered URLs into the canonical endpoint file.
    Commands:
    - `katana -list live.txt -d 3 -kf all -js-crawl -silent > endpoints/endpoints-katana.txt`
    - `katana -list live.txt -d 3 -kf all -js-crawl -headless -silent > endpoints/endpoints-headless.txt` use this where normal crawling misses rendered content.
    - `cat endpoints/endpoints-katana.txt endpoints/endpoints-headless.txt | sort -u > endpoints/endpoints.txt.tmp && mv endpoints/endpoints.txt.tmp endpoints/endpoints.txt`
    - Use only the programme-approved contact header and rate limit; do not copy a header value from another programme.

8. Run the JavaScript reconnaissance methodology.
    - Change into `javascript/` and use `../endpoints/endpoints.txt` as the endpoint input for `Javascript-recon.md`.
    - Merge confirmed JavaScript and source-map endpoints back into `../endpoints/endpoints.txt` with a temporary file, as shown in that guide.
    - Merge confirmed parameter-bearing URLs into `../endpoints/params-urls.txt` and record API, authentication, and client-side context for deep manual review.

9. Build a canonical parameter inventory without overwriting earlier results.
    Commands:
    - `{ grep -E '\?' endpoints/endpoints.txt; gau --subs target.com | grep '='; } | sort -u > endpoints/params-urls.txt.tmp && mv endpoints/params-urls.txt.tmp endpoints/params-urls.txt`
    - `grep -oE '[?&][^=&]+' endpoints/params-urls.txt | cut -c2- | sort -u > endpoints/params.txt`
    - Review URLs before testing: archived URLs and JavaScript findings can require authentication or refer to retired functionality.

10. Scan non-web services only when service scanning is permitted by the programme.
    Commands:
    - `nmap -iL live-hosts.txt -sV --open -T3 -oN vulnerability_scans/nmap-scan.txt`
    - Use default scripts (`-sC`) only where the programme explicitly permits them, because some scripts are intrusive.

11. Run only the automated checks allowed by scope, and keep raw results separate from verified findings.
    Commands:
    - `subzy run --targets live.txt > vulnerability_scans/subdomain-takeover-scan.txt`
    - `gau --subs target.com | gf <installed-pattern-name> > vulnerability_scans/<pattern-name>-urls.txt` use only installed, relevant `gf` patterns.
    - Save each scanner's raw output in `vulnerability_scans/`. Add a result to `vulnerability_scans/vulnerabilitys/` only after manual validation.

12. Identify high-interest subdomains and prepare them for deep review.
    - Prioritise authentication and administration hosts, APIs, GraphQL and WebSocket endpoints, upload and download flows, exposed cloud storage, unusual services, and explicitly in-scope non-production hosts.
    - Create `manual-recon/<interesting_subdomain_name>/` using the structure in the main `README.md` and record why the host was selected in its `notes.md` or `interesting-endpoints.md`.

---

### Deep subdomain attack surface enumeration

This phase examines one authorised subdomain in detail and preserves its context in `manual-recon/<interesting_subdomain_name>/`.

1. Gather the wide-scan results for the chosen subdomain into its manual-recon notes.
    Commands:
    - `grep -F 'sub.target.com' endpoints/endpoints.txt`
    - `grep -F 'sub.target.com' endpoints/params-urls.txt`
    - `grep -F 'sub.target.com' live-with-info.txt`
    - `grep -F 'sub.target.com' domains-manual.txt`
    - `grep -F 'sub.target.com' dorks.txt`
    - `grep -F 'sub.target.com' vulnerability_scans/nmap-scan.txt`
    - `grep -F 'sub.target.com' vulnerability_scans/subdomain-takeover-scan.txt`
    - `grep -F 'sub.target.com' technology/technology-inventory.tsv`
    - `grep -F 'sub.target.com' technology/whatweb.txt`
    - Copy relevant results into `technology-notes.md`, `api-notes.md`, `javascript-review.md`, and `interesting-endpoints.md` rather than leaving them only in terminal output.

2. Build a thorough manual technology profile for the selected host.
    The automatic inventory is only a starting point. Confirm the technologies that affect attack surface and record the evidence, version where exposed, confidence, and date in `technology-notes.md`.
    Commands:
    - `httpx -u https://sub.target.com -title -tech-detect -status-code -follow-redirects -web-server -ip -cdn -asn > technology-httpx.txt`
    - `curl -sS -I -L https://sub.target.com > response-headers.txt` record server, framework, cache, CDN, security, and identity-provider headers.
    - `whatweb -a 3 --no-errors https://sub.target.com > whatweb.txt` use where WhatWeb is installed; compare this result with `technology-httpx.txt` instead of trusting either tool alone.
    - `curl -sS https://sub.target.com/robots.txt > robots.txt` and `curl -sS https://sub.target.com/sitemap.xml > sitemap.xml` to identify documented paths and application components.
    - Use the Wappalyzer extension and browser developer tools to inspect loaded scripts, cookies, local storage, service workers, network requests, API protocols, WebSockets, and third-party domains.
    - Review newly found JavaScript and source maps using `Javascript-recon.md`; package names, build paths, and API clients often identify the framework and backend services.
    - If TLS, DNS, port, or certificate details are relevant to the host, copy the evidence from `live-with-info.txt`, `technology/`, and permitted service scans into `technology-notes.md`.
    Record at least: hostname and final URL; CDN/WAF and hosting clues; server and framework; client-side framework and major libraries; API type and documentation; authentication/identity provider; data, storage, analytics, and payment providers; security headers; relevant cookies; and confidence/source for every important fingerprint.

3. Translate the technology profile into focused manual-recon tasks.
    - Put login, sessions, MFA, OAuth/OIDC, SAML, and token observations in `authentication.md`.
    - Put roles, organisation boundaries, tenant identifiers, and object ownership observations in `authorization.md`.
    - Put API methods, schemas, GraphQL, WebSockets, and request context in `api-notes.md`.
    - Put client-side libraries, browser storage, service workers, source-map findings, and DOM data flows in `javascript-review.md`.
    - Put user journeys, feature flags, integrations, payments, invitations, and state-changing actions in `business-logic.md`.

4. Map authenticated-only functionality and endpoints with testing accounts.
    Create an endpoint and access matrix for each available authorised role, such as anonymous, standard user, organisation member, manager, administrator, and support user. Use only accounts, tenants, and test data that the programme permits. Keep credentials, cookies, bearer tokens, and exported proxy session files out of the recon directory and version control.
    - Capture an unauthenticated baseline, then sign in through the normal application flow and repeat the same user journeys with each authorised role. Use browser developer tools or Burp Suite's proxy/history and crawler with session-handling rules; this preserves normal authentication without placing secrets in commands or shell history.
    - Visit each authorised area normally: account and profile settings, organisation or tenant settings, member management, invitations, billing, support, dashboards, exports, uploads/downloads, API-key management, integrations, webhooks, audit logs, and administrative features. Record every observed page, API request, HTTP method, content type, parameter or body field names, response status, and authentication context.
    - Export proxy history or crawler URLs for each role to separate files such as `endpoints-anonymous.txt`, `endpoints-user.txt`, and `endpoints-admin.txt`. Merge the reviewed, in-scope URL set into `all-endpoints.txt`, retaining the role-specific files as provenance.
    - Create `authenticated-endpoint-matrix.tsv` with at least: URL or route, method, feature, role(s) observed, authentication required, tenant or object context, state-changing (`yes`/`no`/`unknown`), source (browser/proxy/client/spec), and notes. Record API details and request-shape evidence in `api-notes.md`; record roles, object ownership, and cross-tenant observations in `authorization.md`.
    - Compare route and request inventories across roles and tenants. Prioritise routes visible only after login, client-code or specification operations not reached by the UI, admin or support functions, exports, uploads, bulk actions, API-key and webhook management, and endpoints where the same route behaves differently by role or tenant.
    - Build a request baseline before later authorization testing: preserve a harmless representative request, its expected status and response shape, the acting role, target object owner, tenant, and whether it is read-only. Do not replay state-changing requests, modify identifiers, or switch tenants during mapping; move those hypotheses to the separately authorised authorization-testing workflow.
    - Review browser storage, cookies, token-refresh requests, service workers, background polling, WebSocket connections, and hidden feature-flag responses while authenticated. Record endpoint and protocol metadata, but do not copy secret values into `authentication.md`, `api-notes.md`, or screenshots.

5. Discover and document API specifications, schemas, and real-time protocols. Treat every discovered route, field, operation, and event name as a lead; do not invoke mutations, state-changing RPC methods, or WebSocket events merely to test whether they exist. Check common API documentation and specification paths, baselining normal responses first and using the programme's approved rate limit.
    Commands:
    - `for path in /openapi.json /openapi.yaml /swagger.json /swagger.yaml /api-docs /api-docs.json /v1/openapi.json /v2/api-docs /swagger-ui /swagger-ui/index.html /api/swagger.json /docs /redoc; do curl -sS -o "api-spec-$(echo "$path" | tr '/' '_').txt" -w "%{http_code}\t%{content_type}\t%{size_download}\t$path\n" "https://sub.target.com$path"; done > api-spec-probes.tsv`
    - Review `api-spec-probes.tsv` and retain only confirmed documentation or specification responses in `api-notes.md`. Save JSON or YAML specifications using meaningful names, for example `openapi.json` or `swagger.yaml`, before parsing them.
    - For a confirmed OpenAPI or Swagger JSON file, extract the documented paths, methods, parameter names, request-body content types, response codes, schema fields, and security schemes.
    Commands:
    - `jq -r '.paths // {} | to_entries[] | .key as $path | .value | to_entries[] | select(.key | IN("get"; "put"; "post"; "delete"; "patch"; "head"; "options")) | [$path, (.key | ascii_upcase), (.value.operationId // ""), ((.value.parameters // []) | map(.name // "") | join(",")), ((.value.requestBody.content? // {}) | keys | join(",")), ((.value.responses // {}) | keys | join(","))] | @tsv' openapi.json > openapi-operations.tsv`
    - `jq -r '(.components.schemas // .definitions // {}) | to_entries[] | [.key, (.value.type // ""), ((.value.required // []) | join(",")), ((.value.properties // {}) | keys | join(","))] | @tsv' openapi.json > openapi-schema-fields.tsv`
    - `jq -r '(.components.securitySchemes // .securityDefinitions // {}) | to_entries[] | [.key, (.value.type // ""), (.value.scheme // ""), (.value.in // "")] | @tsv' openapi.json > openapi-security-schemes.tsv`
    - Convert the documented paths to absolute URLs, merge them into `all-endpoints.txt`, and record whether each operation is authenticated or potentially state-changing in `api-notes.md`. Do not automatically request every endpoint.
    - For YAML specifications, use an installed YAML-aware parser or review the file manually; do not treat a JSON parser failure as evidence that a specification is invalid.
    - Identify GraphQL from observed requests, JavaScript, technology fingerprints, or documentation. If introspection is explicitly allowed, use a minimal schema query against the confirmed endpoint and save the response as `graphql-schema.json`; otherwise derive operation names, variables, fragments, and types from normal application traffic, persisted-query payloads, and client code.
    Commands:
    - `grep -RniE 'graphql|__schema|__type|query[[:space:]]+[A-Za-z_]|mutation[[:space:]]+[A-Za-z_]|subscription[[:space:]]+[A-Za-z_]' ../../automatic-recon/javascript/js-files/ ../../automatic-recon/javascript/source-maps/ 2>/dev/null > graphql-client-leads.txt`
    - Record GraphQL queries, mutations, subscriptions, variable names, authentication context, and evidence source in `api-notes.md`. Do not execute mutations or subscriptions solely for enumeration.
    - Identify gRPC and protobuf services from response headers (`application/grpc`), client code, mobile applications, public `.proto` files, gateway routes, or documentation. Use server reflection only where explicitly permitted; otherwise record service and method names from passive sources in `api-notes.md`.
    - Identify WebSocket and Socket.IO endpoints from browser developer tools, client code, and observed network traffic. Record the connection URL, handshake headers/cookies, namespaces, event names, message shapes, authentication timing, and whether an event changes state in `api-notes.md`. Do not send arbitrary events to enumerate a live service.
    - Search confirmed specifications and client artifacts for deprecated operations, hidden tags, admin/internal paths, mass-assignment-prone request fields, identifiers, pagination controls, upload/download endpoints, webhooks, and authorization scopes. Copy leads to `interesting-endpoints.md`.

6. Crawl the host again with its known context and run the JavaScript methodology on newly discovered files.
    Commands:
    - `httpx -u https://sub.target.com -title -tech-detect -status-code -follow-redirects -web-server -ip -cdn -asn > technology-httpx.txt`
    - `katana -u https://sub.target.com -d 5 -headless -js-crawl -jsluice -kf all -silent > all-endpoints.txt`
    - Run the Burp Suite crawler using authorised test accounts where applicable and save exported URLs to `all-endpoints.txt`.
    - Merge unique entries from `all-endpoints.txt` into `interesting-endpoints.md` and run `Javascript-recon.md` for newly discovered JavaScript files. Copy high-value API, authentication, and client-side findings to `api-notes.md`, `authentication.md`, and `javascript-review.md`.

7. Review cloud storage.
    Commands:
    - `cd /home/g/Hacking/tools/cloud_enum && uv run python cloud_enum.py -k company_name`
    - `aws s3 ls s3://bucket-name --no-sign-request` list only a confirmed, in-scope bucket.
    - `aws s3 cp s3://bucket-name/object-name ./downloaded-object --no-sign-request` download only an authorised object to a local review directory.
    - Record confirmed buckets, objects, permissions, and scope evidence in `cloud_storage.md`.

8. Perform additional public-information research for the selected host.
    - Record the query, date, source URL, and relevant result in the host's `notes.md`.
    - Confirm that any discovered host, bucket, or third-party service is in scope before interacting with it.

9. Enumerate endpoint and API parameters only after baselining the response and confirming the method is safe.
    Commands:
    - `ffuf -w "/home/g/Hacking/hacking-map/Bug bounty methodology/Recon/wordlists/raft-large-words.txt" -u "https://sub.target.com/page?FUZZ=test" -ac -rate 20 -of json -o parameter-name-fuzz.json`
    - `ffuf -w "/home/g/Hacking/hacking-map/Bug bounty methodology/Recon/wordlists/large.txt" -u "https://sub.target.com/api" -X POST -H "Content-Type: application/json" -d '{"FUZZ":"test"}' -ac -rate 20 -of json -o api-parameter-fuzz.json` run only against an authorised, non-state-changing request.
    - Record the baseline, authentication context, rate limit, endpoint, method, and parameter in `api-notes.md`. Manually validate response differences before treating them as findings.
