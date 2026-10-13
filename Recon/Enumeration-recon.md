# Wide subdomain enumeration

### Key concepts

The goal of wide subdomain enumeration is to discover as much authorised attack surface as possible, especially hidden development, staging, API, and administrative subdomains, and every root domain, IP range, and content path that the programme owns. Breadth is what turns a single medium-severity bug into a critical: the more of the real surface you see, the more attack vectors you can chain. Keep raw results, but use one canonical file for each type of result so that later stages do not miss earlier discoveries.

Coverage is organised so that every major attack vector has a recon source feeding it: root/ASN discovery and passive/brute/permutation enumeration feed subdomain takeover and forgotten-host bugs; DNS-record review feeds dangling-record and third-party-service takeovers; content and parameter discovery feed injection, access-control, and file-handling bugs; historical URLs feed retired-but-live endpoints; technology and WAF fingerprinting feed CVE and filter-bypass work; and the JavaScript methodology feeds API, authentication, and client-side testing.

---

### Before starting

1. Read the program rules and create the target directory structure described in the main `README.md`.
    - Record in-scope domains, excluded domains, allowed testing types, rate limits, and the required contact or user-agent header in `scope.md`, `out-of-scope.md`, and `recon/automatic-recon/notes.md`.
    - Run the commands below from `target.com/recon/automatic-recon/` and replace `target.com` with the authorised root domain.
    - Do not scan third-party hosts, cloud storage, or non-production systems unless they are explicitly in scope. Active steps below (virtual-host fuzzing, crawling, content and parameter discovery, `nuclei`, `nmap`) must only be run against hosts you have confirmed are in scope, at the programme's approved rate limit, with the programme's contact header.
    - Several steps use tools that are optional additions to the core set. Where a tool is marked *if installed*, run it only when it is present; the core methodology still works without it. See the setup tables in `README.md` for install links.

### Steps for enumerating subdomains and their attack surface

1. Discover every root domain, organisation asset, and IP range the programme owns before enumerating subdomains. Wildcard programmes often own more root domains, acquisitions, and netblocks than the single seed they advertise; missing these is the most common way to miss the whole surface. Confirm each discovered asset is in scope before touching it.
    Commands:
    - `amass intel -org "Company Name" -silent > asn-org.txt` map the organisation to its ASNs from registry data.
    - `echo "AS12345" | asnmap -silent | mapcidr -silent | sort -u > cidrs.txt` *if installed* expand an in-scope ASN to its CIDR ranges.
    - `dnsx -ptr -l cidrs.txt -resp-only -silent | sort -u > reverse-dns.txt` resolve in-scope ranges to hostnames via reverse DNS; review these for additional root domains.
    - `whois target.com` and reverse-WHOIS / favicon services (record the query and result in `dorks.txt`) to find sibling domains registered by the same organisation. For favicon correlation, compute the icon's murmur3 hash and search Shodan `http.favicon.hash:<hash>` to find hosts serving the same application.
    - Record every confirmed in-scope root domain in `roots.txt` (one per line) and run steps 2–5 for each root. Add newly discovered acquisitions and brands to your list of programmes to check against scope.

2. Run passive subdomain discovery from as many independent sources as possible, then merge. Out-of-the-box `subfinder` returns the same names everyone else finds; adding `amass` passive mode and certificate transparency widens coverage. Configure `subfinder`'s provider API keys (`~/.config/subfinder/provider-config.yaml`) so passive sources such as SecurityTrails, Censys, and VirusTotal are actually queried.
    Commands:
    - `subfinder -d target.com -all -silent | sort -u > domains-subfinder.txt`
    - `amass enum -passive -d target.com -silent 2>/dev/null | sort -u > domains-amass.txt`
    - `curl -s "https://crt.sh/?q=%25.target.com&output=json" | jq -r '.[].name_value' | sed 's/\*\.//' | sort -u > domains-crtsh.txt` certificate-transparency names.

3. Brute-force additional subdomains against a resolver list, then merge all passive and brute sources into the canonical `domains.txt` file.
    Commands:
    - `dnsx -silent -d target.com -w "/home/g/Hacking/hacking-map/Bug-bounty-methodology/Recon/wordlists/namelist.txt" > domains-dnsx.txt`
    - `cat domains-subfinder.txt domains-amass.txt domains-crtsh.txt domains-dnsx.txt domains-manual.txt 2>/dev/null | sed 's/\.$//' | sort -u > domains.txt.tmp && mv domains.txt.tmp domains.txt`
    - Add reviewed results from search engines or other manual sources to `domains-manual.txt` before merging them.

4. Generate and resolve permutations of known subdomains. Environments such as `api-dev`, `admin-staging`, and `internal-2` are frequently reachable but absent from passive sources.
    Commands:
    - `alterx -l domains.txt -silent 2>/dev/null | dnsx -silent | sort -u > domains-permutations.txt` *if installed* permute known names and keep only those that resolve.
    - `cat domains.txt domains-permutations.txt | sort -u > domains.txt.tmp && mv domains.txt.tmp domains.txt`

5. Resolve the full domain list and capture DNS records. `CNAME` records that point to unclaimed third-party services, and `NS`/`MX` records pointing to deprovisioned providers, are direct subdomain-takeover leads; `TXT` records disclose SaaS providers and verification tokens worth noting.
    Commands:
    - `dnsx -l domains.txt -silent -a -aaaa -cname -txt -resp -json > dns-records.jsonl`
    - `jq -r 'select(.cname != null) | [.host, (.cname | join(","))] | @tsv' dns-records.jsonl | sort -u > cname-records.tsv` review these for dangling records pointing to S3, GitHub Pages, Heroku, Azure, and similar services, and copy takeover candidates to `notes.md`.
    - `jq -r 'select(.txt != null) | [.host, (.txt | join(" "))] | @tsv' dns-records.jsonl | sort -u > txt-records.tsv` review for third-party services and providers.

6. Enumerate virtual hosts.
    Determine the default response size or word count first, then filter that baseline to reduce wildcard and default-host false positives.
    Commands:
    - `ffuf -w "/home/g/Hacking/hacking-map/Bug-bounty-methodology/Recon/wordlists/subdomains-top1million-110000.txt" -u "https://target.com/" -H "Host: FUZZ.target.com" -ac -rate 20 -of json -o ffuf-vhosts.json`
    - Review `ffuf-vhosts.json`, validate candidate hostnames with DNS or `httpx`, then add confirmed names to `domains-manual.txt` and merge them into `domains.txt`.

7. Use search engines, public sources, and internet-wide scan databases to find relevant company information, exposed hosts, and services. Save the query, date, source URL, and relevant result in `dorks.txt`; do not treat search snippets or scan-database records as confirmed findings until you have verified them in scope.
    Tools:
    - https://pentest-tools.com/information-gathering/google-hacking for example search operators.
    - https://ahrefs.com/blog/google-advanced-search-operators/ for additional search operators.
    - Shodan (`org:"Company Name"`, `ssl.cert.subject.cn:"target.com"`, `http.favicon.hash:<hash>`) and Censys (`services.tls.certificates.leaf_data.subject.common_name:target.com`) to find exposed hosts, non-standard ports, and services not reachable from DNS enumeration. Use the web interfaces or the `shodan` / `censys` CLIs *if installed and configured*, and record only in-scope results.
    - GitHub and GitLab code search for `target.com`, internal hostnames, and API paths to surface additional subdomains and leaked references; move confirmed secrets to the appropriate testing workflow.

8. Resolve and probe the complete domain list to identify live web services.
    Commands:
    - `httpx -l domains.txt -silent -sc -ip -tech-detect -follow-redirects > live-with-info.txt`
    - `httpx -l domains.txt -silent -follow-redirects | sort -u > live.txt`
    - `sed -E 's#https?://([^/:]+).*#\1#' live.txt | sort -u > live-hosts.txt`
    - Keep `domains.txt` as all discovered in-scope names and `live.txt` as the canonical list of reachable web URLs.

9. Build an automatic technology inventory for every live web service, fingerprint edge protections, and capture screenshots for visual triage.
    This creates a broad view of delivery infrastructure, server software, frameworks, client-side libraries, API protocols, identity providers, storage providers, and security controls. Knowing the WAF and stack up front tells you which CVEs and which filter-bypass techniques are worth trying. Treat automated fingerprints as leads and verify important ones during manual review.
    Commands:
    - `mkdir -p technology`
    - `httpx -l live.txt -silent -json -sc -title -tech-detect -web-server -ip -cdn -asn -follow-redirects > technology/httpx.jsonl`
    - `jq -r '[.url, ((.status_code // "") | tostring), (.title // ""), (.webserver // ""), ((.tech // []) | join(",")), (.cdn_name // ""), ((.asn // "") | tostring)] | @tsv' technology/httpx.jsonl > technology/technology-inventory.tsv`
    - `whatweb --input-file live.txt --no-errors -a 1 > technology/whatweb.txt` use as a second, low-impact fingerprinting source if WhatWeb is installed.
    - `wafw00f -i live.txt -o technology/wafw00f.txt` *if installed* identify the WAF/CDN in front of each host so later bypass work is targeted.
    - `gowitness scan file -f live.txt --screenshot-path technology/screenshots/ 2>/dev/null` *if installed* screenshot every live host so you can triage the surface visually and spot login pages, admin panels, default installs, and error pages quickly.
    - `grep -RniE 'graphql|graphiql|swagger|openapi|/api/|socket\.io|wss://|oauth|openid|saml|stripe|paypal|s3|firebase|cloudfront|vercel|netlify' technology/ > technology/technology-leads.txt`
    - Review `technology/technology-inventory.tsv` and `technology/technology-leads.txt`; record promising hosts and the evidence in `notes.md` before moving them to manual recon.

10. Crawl live web applications and merge all discovered URLs into the canonical endpoint file.
    Commands:
    - `katana -list live.txt -d 3 -kf all -js-crawl -jsluice -silent > endpoints/endpoints-katana.txt`
    - `katana -list live.txt -d 3 -kf all -js-crawl -headless -silent > endpoints/endpoints-headless.txt` use this where normal crawling misses rendered content.
    - `cat endpoints/endpoints-katana.txt endpoints/endpoints-headless.txt | sort -u > endpoints/endpoints.txt.tmp && mv endpoints/endpoints.txt.tmp endpoints/endpoints.txt`
    - Use only the programme-approved contact header and rate limit; do not copy a header value from another programme.

11. Pull historical URLs from public archives and merge them into the endpoint inventory. Archived URLs frequently reveal retired-but-still-live endpoints, old API versions, and parameters that no longer appear in the live application — all high-value attack surface that crawling alone misses.
    Commands:
    - `gau --subs target.com | sort -u > endpoints/urls-gau.txt`
    - `waymore -i target.com -mode U -oU endpoints/urls-waymore.txt 2>/dev/null` *if installed* as a second archive source (Wayback, Common Crawl, URLScan, VirusTotal).
    - `cat endpoints/endpoints.txt endpoints/urls-gau.txt endpoints/urls-waymore.txt 2>/dev/null | sort -u > endpoints/endpoints.txt.tmp && mv endpoints/endpoints.txt.tmp endpoints/endpoints.txt`
    - Archived URLs may require authentication, may be out of scope, or may refer to retired functionality; review before testing.

12. Run the JavaScript reconnaissance methodology.
    - Change into `javascript/` and use `../endpoints/endpoints.txt` as the endpoint input for `Javascript-recon.md`.
    - Merge confirmed JavaScript and source-map endpoints back into `../endpoints/endpoints.txt` with a temporary file, as shown in that guide.
    - Merge confirmed parameter-bearing URLs into `../endpoints/params-urls.txt` and record API, authentication, and client-side context for deep manual review.

13. Discover content and files that are not linked from the application or present in archives. Directory and file brute-forcing surfaces admin panels, backups, exposed version-control directories, configuration files, API documentation, and debug endpoints — the paths most likely to lead to high-severity findings. Run this only against in-scope hosts, at the approved rate limit; prioritise interesting hosts from the technology inventory and screenshots rather than brute-forcing every live host.
    Commands:
    - `mkdir -p content`
    - `ffuf -w "/home/g/Hacking/hacking-map/Bug-bounty-methodology/Recon/wordlists/sensitive-files.txt" -u "https://sub.target.com/FUZZ" -ac -rate 20 -of json -o content/sensitive-files-sub.json` check the curated sensitive-files list first (`.git/config`, `.env`, backups, dumps).
    - `ffuf -w "/home/g/Hacking/hacking-map/Bug-bounty-methodology/Recon/wordlists/raft-large-words.txt" -u "https://sub.target.com/FUZZ" -ac -rate 20 -of json -o content/dirs-sub.json` broad directory and file discovery.
    - `nuclei -l live.txt -tags exposure,config,backup,logs -rl 20 -o vulnerability_scans/nuclei-exposures.txt` *run only if automated scanning is permitted by the programme* to find exposed panels, config files, and information disclosure across the surface.
    - Manually verify each candidate (some `200` responses are soft-404s or wildcard pages) and record confirmed paths in `content/interesting-content.txt` with the host, path, status, and why it is interesting.

14. Build a canonical parameter inventory and discover hidden parameters without overwriting earlier results. Hidden parameters feed injection, access-control, and mass-assignment testing.
    Commands:
    - `{ grep -E '\?' endpoints/endpoints.txt; gau --subs target.com | grep '='; } | sort -u > endpoints/params-urls.txt.tmp && mv endpoints/params-urls.txt.tmp endpoints/params-urls.txt`
    - `grep -oE '[?&][^=&]+' endpoints/params-urls.txt | cut -c2- | sort -u > endpoints/params.txt`
    - `arjun -u "https://sub.target.com/endpoint" -oT endpoints/arjun-params.txt 2>/dev/null` *if installed* discover parameters the application accepts but does not advertise; run only against an authorised, non-state-changing endpoint at the approved rate limit.
    - Review URLs before testing: archived URLs and JavaScript findings can require authentication or refer to retired functionality.

15. Scan non-web services only when service scanning is permitted by the programme.
    Commands:
    - `nmap -iL live-hosts.txt -sV --open -T3 -oN vulnerability_scans/nmap-scan.txt`
    - Use default scripts (`-sC`) only where the programme explicitly permits them, because some scripts are intrusive.

16. Run only the automated checks allowed by scope, and keep raw results separate from verified findings. Active scanners such as `nuclei` must only run against in-scope hosts, at the approved rate limit, with the programme's contact header.
    Commands:
    - `subzy run --targets live.txt > vulnerability_scans/subdomain-takeover-scan.txt`
    - `nuclei -l live.txt -tags takeover,cve,misconfiguration,default-login,exposure -severity medium,high,critical -rl 20 -o vulnerability_scans/nuclei.txt` broad templated checks for takeovers, known CVEs, misconfigurations, and default credentials; use the local templates at `/home/g/Hacking/tools/nuclei-templates/` and update them before running.
    - `gau --subs target.com | gf <installed-pattern-name> > vulnerability_scans/<pattern-name>-urls.txt` use only installed, relevant `gf` patterns (for example `xss`, `sqli`, `ssrf`, `redirect`, `lfi`, `idor`) to bucket URLs by likely vulnerability class.
    - Save each scanner's raw output in `vulnerability_scans/`. Add a result to `vulnerability_scans/vulnerabilitys/` only after manual validation.

17. Identify high-interest subdomains and prepare them for deep review.
    - Prioritise authentication and administration hosts, APIs, GraphQL and WebSocket endpoints, upload and download flows, exposed cloud storage, unusual services and ports, hosts with distinctive technology or WAF fingerprints, subdomain-takeover leads from the DNS-record review, and explicitly in-scope non-production hosts.
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
    - `grep -F 'sub.target.com' cname-records.tsv`
    - `grep -F 'sub.target.com' content/interesting-content.txt`
    - `grep -F 'sub.target.com' vulnerability_scans/nmap-scan.txt`
    - `grep -F 'sub.target.com' vulnerability_scans/subdomain-takeover-scan.txt`
    - `grep -F 'sub.target.com' vulnerability_scans/nuclei.txt`
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
    - Identify GraphQL from observed requests, JavaScript, technology fingerprints, or documentation. Fingerprint the engine to know which introspection and batching behaviours to expect: `python3 /home/g/Hacking/tools/graphw00f/main.py -d -f -t https://sub.target.com/graphql` *if installed*. If introspection is explicitly allowed, use a minimal schema query against the confirmed endpoint and save the response as `graphql-schema.json`; otherwise derive operation names, variables, fragments, and types from normal application traffic, persisted-query payloads, and client code.
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

7. Run targeted deep content, parameter, and safe configuration checks against the authorised host. Baseline normal responses first and use the programme's approved rate limit. These probe attack surface directly, so confirm the host is in scope and the checks are permitted before running them.
    Commands:
    - `ffuf -w "/home/g/Hacking/hacking-map/Bug-bounty-methodology/Recon/wordlists/raft-large-words.txt" -u "https://sub.target.com/FUZZ" -ac -rate 20 -recursion -recursion-depth 1 -of json -o content-deep.json` deeper, recursive content discovery on the single host.
    - `ffuf -w "/home/g/Hacking/hacking-map/Bug-bounty-methodology/Recon/wordlists/raft-large-words.txt" -u "https://sub.target.com/api/FUZZ" -ac -rate 20 -of json -o content-api.json` enumerate undocumented API routes under a known base path.
    - `python3 /home/g/Hacking/tools/CORScanner/cors_scan.py -u https://sub.target.com -v > cors-scan.txt` check for permissive cross-origin configurations that enable credentialed cross-origin reads.
    - For endpoints that return `401`/`403`, test documented access-control-bypass techniques with `bash /home/g/Hacking/tools/bypass-403/bypass-403.sh https://sub.target.com/protected-path` and record any that return `200`; validate manually before treating as a finding.
    - Record baselines, confirmed paths, and access-control observations in `interesting-endpoints.md`, `api-notes.md`, and `authorization.md`.

8. Review cloud storage.
    Commands:
    - `cd /home/g/Hacking/tools/cloud_enum && uv run python cloud_enum.py -k company_name`
    - `aws s3 ls s3://bucket-name --no-sign-request` list only a confirmed, in-scope bucket.
    - `aws s3 cp s3://bucket-name/object-name ./downloaded-object --no-sign-request` download only an authorised object to a local review directory.
    - Record confirmed buckets, objects, permissions, and scope evidence in `cloud_storage.md`.

9. Perform additional public-information research for the selected host.
    - Record the query, date, source URL, and relevant result in the host's `notes.md`.
    - Confirm that any discovered host, bucket, or third-party service is in scope before interacting with it.

10. Enumerate endpoint and API parameters only after baselining the response and confirming the method is safe.
    Commands:
    - `ffuf -w "/home/g/Hacking/hacking-map/Bug-bounty-methodology/Recon/wordlists/raft-large-words.txt" -u "https://sub.target.com/page?FUZZ=test" -ac -rate 20 -of json -o parameter-name-fuzz.json`
    - `ffuf -w "/home/g/Hacking/hacking-map/Bug-bounty-methodology/Recon/wordlists/large.txt" -u "https://sub.target.com/api" -X POST -H "Content-Type: application/json" -d '{"FUZZ":"test"}' -ac -rate 20 -of json -o api-parameter-fuzz.json` run only against an authorised, non-state-changing request.
    - `arjun -u "https://sub.target.com/endpoint" -oT arjun-params.txt 2>/dev/null` *if installed* discover hidden parameters the endpoint accepts; run only against authorised, non-state-changing endpoints.
    - Record the baseline, authentication context, rate limit, endpoint, method, and parameter in `api-notes.md`. Manually validate response differences before treating them as findings.
