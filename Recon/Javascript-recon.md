# JavaScript beautifying and reconnaissance

The aim of javascript analysis is to discover endpoints, parameters, exposed configuration, secrets, and functionality developers did not intend people to know about. Modern single-page applications push most of the routing, API contract, and feature logic into the client, so thorough JavaScript review often exposes more attack surface than crawling the rendered site. Only collect and test assets that are in the authorised target scope.

---
### Steps for javaScript analysis

Run these commands from `target.com/recon/automatic-recon/javascript/`. The canonical endpoint files are in `../endpoints/`, and all JavaScript-specific results stay in this directory.

1. Gather all JavaScript endpoints and source maps from the crawl, the live page, and public archives. Archived (historical) bundles matter: old builds frequently still contain secrets, API endpoints, and feature references that were removed from the current bundle but remain reachable on the backend.
    Commands:
    - `grep -E '\.js($|\?)' ../endpoints/endpoints.txt | sed -E 's/[?#].*$//' | sort -u > js-files.txt`
    - `grep -E '\.map($|\?)' ../endpoints/endpoints.txt | sed -E 's/[?#].*$//' | sort -u > source-maps.txt`
    - `curl -s https://target.com | grep -oE 'src="[^"]+\.js[^"]*"' | cut -d'"' -f2 | sed -E 's#^//#https://#; s#^/#https://target.com/#; s/[?#].*$//' | sort -u > js-files-extra.txt`
    - `gau --subs target.com | grep -E '\.js($|\?)' | sed -E 's/[?#].*$//' | sort -u > js-files-archived.txt` collect historical JavaScript URLs from public archives.
    - `cat js-files.txt js-files-extra.txt js-files-archived.txt | sort -u > js-files.tmp && mv js-files.tmp js-files.txt` merge the results when finished.
    - URLs that are relative to a page or bundle must be converted to absolute URLs before downloading them.

2. Download all javascript files you have collected and check that they are live javascript responses.
    Commands:
    - `httpx -l js-files.txt -silent -mc 200,206,304 > live-js-files.txt`
    - `wget -i live-js-files.txt -P js-files`
    - `find js-files -type f -exec file {} \; | grep -viE 'HTML document|empty'` check for 404 pages, redirects, and other non-javascript responses.
    - `grep -RhoE 'sourceMappingURL=[^[:space:]]+' js-files/ | sed 's/sourceMappingURL=//' | sort -u > source-maps-extra.txt`
    - Resolve relative source-map URLs against the javascript file they came from, then merge them with `source-maps.txt`.

3. Deduplicate javascript files by content before reviewing them.
    Commands:
    - `find js-files -type f -name '*.js' -exec sha256sum {} + | sort > js-file-hashes.txt`
    - `awk '{print $1}' js-file-hashes.txt | uniq -d` lists hashes that occur more than once.
    - Keep one copy of identical files and note the duplicate file names so that CDN and subdomain copies are not reviewed repeatedly.

4. Beautify all javascript files you have collected.
    Commands:
    - `find js-files -type f -name '*.js' -exec sh -c 'js-beautify "$1" -o "$1.tmp" && mv "$1.tmp" "$1"' _ {} \;`

5. Find dynamically loaded javascript chunks. Modern applications often load extra chunks only after a user visits a page or uses a feature, and a webpack runtime often lists every lazy-loaded chunk by name even before it is requested.
    Commands:
    - Open the authorised target in a browser or intercepting proxy and use the application normally.
    - In the Network tab, filter for `JS`, copy or export the requested javascript URLs, and save them to `dynamic-js-files.txt`.
    - `grep -RhoE '"[0-9]+":"[a-f0-9]+"' js-files/ | sort -u > webpack-chunk-map.txt` extract the chunk-id-to-hash map from the webpack runtime; reconstruct the chunk URLs against the bundle's `publicPath` and add reachable ones to `dynamic-js-files.txt`.
    - `cat js-files.txt dynamic-js-files.txt | sort -u > js-files.tmp && mv js-files.tmp js-files.txt` merge the results, then download and beautify the new files.

6. Extract endpoints and links from the javascript files. Use more than one extractor, because each parser misses different patterns; `linkfinder` uses regular expressions while `jsluice` (via Katana) parses the JavaScript to find URLs, paths, and secrets.
    Commands:
    - `cd /home/g/Hacking/tools/LinkFinder && source venv/bin/activate` activates the python virtual environment for linkfinder.
    - `find js-files -type f -name '*.js' -print0 | xargs -0 -I{} python3 linkfinder.py -i '{}' -o cli | sort -u > links.txt`
    - `deactivate` leave the linkfinder virtual environment before running other tools.
    - `katana -jsluice -list live-js-files.txt -silent | sort -u >> links.txt` parse the same files with jsluice for endpoints and paths linkfinder's regexes miss, then re-sort with `sort -u links.txt -o links.txt`.
    - `{ cat ../endpoints/endpoints.txt; httpx -l links.txt -silent; } | sort -u > ../endpoints/endpoints.txt.tmp && mv ../endpoints/endpoints.txt.tmp ../endpoints/endpoints.txt` merge live results into the canonical endpoint file.
    - Keep `links.txt` as the full discovered list; it can include relative paths and endpoints that need authentication.

7. Inspect how the application builds API requests and record the useful context.
    Commands:
    - `grep -RniE 'fetch\(|axios\.|XMLHttpRequest|\.ajax\(' js-files/ > api-request-code.txt`
    - `grep -RniE 'method:|headers:|Authorization|Content-Type|params:|data:|body:' js-files/ >> api-request-code.txt`
    - Record the source file, line number, URL or path, HTTP method, parameter names, headers, and whether authentication appears required in `api-findings.txt`.

8. Search javascript files for secrets with a dedicated scanner and with targeted greps. A scanner such as `trufflehog` finds and can verify many high-entropy and known-format credentials automatically; the greps catch application-specific references the scanner does not know about. Do not use exposed credentials outside the authorised scope, and never commit recovered secrets to version control.
    Commands:
    - `trufflehog filesystem js-files/ --results=verified,unknown --json > trufflehog-js.json` scan the downloaded bundles for known-format and verified secrets.
    - `find js-files -type f -name '*.js' -exec jsluice secrets {} + > jsluice-secrets.txt 2>/dev/null` *if the standalone `jsluice` binary is installed* as a second automated source that parses the JavaScript rather than pattern-matching it.
    - `grep -RniE 'token|jwt|bearer|authorization|oauth|refresh' js-files/`
    - `grep -RniE 'secret|apikey|api_key|password|private.?key|credential|aws_|s3_|gcp|firebase|sentry|mapbox|stripe|twilio|slack|sendgrid|algolia' js-files/`
    - `grep -RniE 'admin|dashboard|manage|internal|isAdmin|role|permission|feature.?flag|entitlement' js-files/`
    - `grep -RniE 'debug|test|dev|staging|sandbox|localhost|127\.0\.0\.1' js-files/`
    - `grep -RniE 'graphql|query|mutation|ws://|wss://|socket\.io|upload|download|s3|bucket|presign|redirect|returnUrl|callback|next' js-files/`
    - Save interesting findings to `secrets.txt` and record the source file and line number. Verify each candidate manually before treating it as a finding; scanners produce false positives and many "keys" are public client identifiers.

9. Identify third-party and internal package names for dependency-confusion leads. Bundles and source maps often reveal the names of internal or scoped npm packages; if a private package name is unclaimed on the public registry, it may be a dependency-confusion candidate. Confirm ownership and scope before acting on this.
    Commands:
    - `grep -RhoE '@[a-z0-9._-]+/[a-z0-9._-]+' js-files/ source-maps/ 2>/dev/null | sort -u > package-names.txt` collect scoped package names referenced in the client.
    - Review `package-names.txt` for internal-looking scopes and record any that appear unpublished on the public registry as leads in `secrets.txt` or `../notes.md`.

10. Review client-side code that handles untrusted browser data, covering the sinks used by common frameworks. Compare each source of attacker-influenced data against the sinks it can reach.
    Commands:
    - `grep -RniE 'innerHTML|outerHTML|insertAdjacentHTML|document\.write|eval\(|setTimeout\(|setInterval\(|new Function|\bFunction\(|import\(' js-files/ > client-side-sinks.txt`
    - `grep -RniE 'dangerouslySetInnerHTML|v-html|srcdoc|\.html\(|\.append\(|\.after\(|\.before\(|\.prepend\(|\$\(' js-files/ >> client-side-sinks.txt` framework and jQuery HTML sinks.
    - `grep -RniE 'location\.|location\.hash|location\.search|document\.cookie|document\.referrer|window\.name|postMessage|addEventListener\(.?.message' js-files/ > client-side-sources.txt`
    - `grep -RniE 'addEventListener\(.?.message|onmessage' js-files/ > postmessage-handlers.txt` list message-event handlers and check each for a missing or weak `event.origin` validation.
    - Compare the sources and sinks manually and record any data flow worth testing in `client-side-findings.txt`.

11. Download and inspect source maps, and reconstruct the original source tree where possible.
    Source maps can expose original unminified source files, route names, comments, internal paths, and complete original source code used to build a bundled javascript file.
    Commands:
    - `wget -i source-maps.txt -P source-maps`
    - `find source-maps -type f -name '*.map' -exec jq -r '.sources[]? // empty' {} \; | sort -u > source-map-sources.txt`
    - `find source-maps -type f -name '*.map' -exec jq -r '.sourcesContent[]? // empty' {} \; > source-map-source-code.txt`
    - For any map containing `sourcesContent`, reconstruct the original file tree to disk so it can be reviewed and grepped like a normal codebase: `for m in source-maps/*.map; do python3 -c "import json,os,sys; d=json.load(open(sys.argv[1])); [ (os.makedirs(os.path.dirname('reconstructed/'+s.lstrip('/.')) or '.', exist_ok=True), open('reconstructed/'+s.lstrip('/.'),'w').write(c)) for s,c in zip(d.get('sources',[]), d.get('sourcesContent') or []) if c ]" "$m"; done` rebuilds the source tree under `reconstructed/`. The `unwebpack-sourcemap` or `sourcemapper` tools do the same job if you prefer a dedicated utility.
    - `grep -RniE 'token|jwt|bearer|authorization|oauth|secret|apikey|password|admin|internal|debug|staging' source-maps/ reconstructed/ 2>/dev/null`
    - `grep -RaoE '(/[^"[:space:]\\]+|https?://[^"[:space:]\\]+)' source-maps/ | sort -u > source-map-endpoints.txt`
    - `trufflehog filesystem reconstructed/ --results=verified,unknown --json >> trufflehog-js.json 2>/dev/null` scan reconstructed source for secrets the minified bundle obscured.
    - `cat ../endpoints/endpoints.txt source-map-endpoints.txt | sort -u > ../endpoints/endpoints.txt.tmp && mv ../endpoints/endpoints.txt.tmp ../endpoints/endpoints.txt`
    - Review interesting reconstructed files manually; they often reveal backend route definitions, permission checks, and internal service names.

12. Triage all findings before moving to testing.
    Commands:
    - `sort -u ../endpoints/endpoints.txt -o ../endpoints/endpoints.txt`
    - `{ cat ../endpoints/params-urls.txt; grep -E '\?' ../endpoints/endpoints.txt; } | sort -u > ../endpoints/params-urls.txt.tmp && mv ../endpoints/params-urls.txt.tmp ../endpoints/params-urls.txt`
    - Add each useful endpoint or finding to `api-findings.txt`, `client-side-findings.txt`, or `secrets.txt` with its source file and line number.
    - Prioritise undocumented API routes, privileged features, upload and download flows, authentication code, GraphQL and WebSocket endpoints, exposed secrets, source-map-reconstructed backend logic, dependency-confusion leads, and non-production hosts that are explicitly in scope.
    - Validate interesting findings safely through normal authorised testing and feed confirmed endpoints back into the main recon process.
