# JavaScript beautifying and reconnaissance

The aim of javascript analysis is to discover endpoints, parameters, exposed configuration, and functionality developers did not intend people to know about. Only collect and test assets that are in the authorised target scope.

---
### Steps for javaScript analysis

Run these commands from `target.com/recon/automatic-recon/javascript/`. The canonical endpoint files are in `../endpoints/`, and all JavaScript-specific results stay in this directory.

1. Gather all endpoints with `.js` in them and gather all source maps you can find that end in `.map`.
    Commands:
    - `grep -E '\.js($|\?)' ../endpoints/endpoints.txt | sed -E 's/[?#].*$//' | sort -u > js-files.txt`
    - `grep -E '\.map($|\?)' ../endpoints/endpoints.txt | sed -E 's/[?#].*$//' | sort -u > source-maps.txt`
    - `curl -s https://target.com | grep -oE 'src="[^"]+\.js[^"]*"' | cut -d'"' -f2 | sed -E 's#^//#https://#; s#^/#https://target.com/#; s/[?#].*$//' | sort -u > js-files-extra.txt`
    - `cat js-files.txt js-files-extra.txt | sort -u > js-files.tmp && mv js-files.tmp js-files.txt` merge the results when finished.
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

5. Find dynamically loaded javascript chunks. Modern applications often load extra chunks only after a user visits a page or uses a feature.
    Commands:
    - Open the authorised target in a browser or intercepting proxy and use the application normally.
    - In the Network tab, filter for `JS`, copy or export the requested javascript URLs, and save them to `dynamic-js-files.txt`.
    - `cat js-files.txt dynamic-js-files.txt | sort -u > js-files.tmp && mv js-files.tmp js-files.txt` merge the results, then download and beautify the new files.

6. Extract extra links from the javascript files using `linkfinder`.
    Commands:
    - `cd /home/g/Hacking/tools/LinkFinder && source venv/bin/activate` activates the python virtual environment for linkfinder.
    - `find js-files -type f -name '*.js' -print0 | xargs -0 -I{} python3 linkfinder.py -i '{}' -o cli | sort -u > links.txt`
    - `{ cat ../endpoints/endpoints.txt; httpx -l links.txt -silent; } | sort -u > ../endpoints/endpoints.txt.tmp && mv ../endpoints/endpoints.txt.tmp ../endpoints/endpoints.txt` merge live results into the canonical endpoint file.
    - Keep `links.txt` as the full discovered list; it can include relative paths and endpoints that need authentication.

7. Inspect how the application builds API requests and record the useful context.
    Commands:
    - `grep -RniE 'fetch\(|axios\.|XMLHttpRequest|\.ajax\(' js-files/ > api-request-code.txt`
    - `grep -RniE 'method:|headers:|Authorization|Content-Type|params:|data:|body:' js-files/ >> api-request-code.txt`
    - Record the source file, line number, URL or path, HTTP method, parameter names, headers, and whether authentication appears required in `api-findings.txt`.

8. Search javascript files for secrets, privileged features, third-party services, and environment references.
    Commands:
    - `grep -RniE 'token|jwt|bearer|authorization|oauth|refresh' js-files/`
    - `grep -RniE 'secret|apikey|api_key|password|private.?key|credential' js-files/`
    - `grep -RniE 'admin|dashboard|manage|internal|isAdmin|role|permission|feature.?flag' js-files/`
    - `grep -RniE 'debug|test|dev|staging|sandbox|localhost' js-files/`
    - `grep -RniE 'graphql|query|mutation|ws://|wss://|socket\.io|upload|download|s3|bucket|presign|redirect|returnUrl|callback|next' js-files/`
    - Save interesting findings to `secrets.txt` and record the source file and line number. Do not use exposed credentials outside the authorised scope.

9. Review client-side code that handles untrusted browser data.
    Commands:
    - `grep -RniE 'innerHTML|outerHTML|insertAdjacentHTML|document\.write|eval\(|setTimeout\(|setInterval\(' js-files/ > client-side-sinks.txt`
    - `grep -RniE 'location\.|location\.hash|location\.search|document\.cookie|postMessage|addEventListener\(.message' js-files/ > client-side-sources.txt`
    - Compare the sources and sinks manually and record any data flow worth testing in `client-side-findings.txt`.

10. Download and inspect source maps.
    Source maps can expose original unminified source files, route names, comments, and internal paths used to build a bundled javascript file.
    Commands:
    - `wget -i source-maps.txt -P source-maps`
    - `grep -RniE 'sourcesContent|sources|webpack|api|admin|internal|debug|staging' source-maps/`
    - `find source-maps -type f -name '*.map' -exec jq -r '.sources[]? // empty' {} \; | sort -u > source-map-sources.txt`
    - `find source-maps -type f -name '*.map' -exec jq -r '.sourcesContent[]? // empty' {} \; > source-map-source-code.txt`
    - `grep -RniE 'token|jwt|bearer|authorization|oauth|secret|apikey|password|admin|internal|debug|staging' source-maps/`
    - `grep -RaoE '(/[^"[:space:]\\]+|https?://[^"[:space:]\\]+)' source-maps/ | sort -u > source-map-endpoints.txt`
    - `cat ../endpoints/endpoints.txt source-map-endpoints.txt | sort -u > ../endpoints/endpoints.txt.tmp && mv ../endpoints/endpoints.txt.tmp ../endpoints/endpoints.txt`
    - Check whether the map contains `sourcesContent`. If it does, it may contain complete original source code. Review interesting files manually.

11. Triage all findings before moving to testing.
    Commands:
    - `sort -u ../endpoints/endpoints.txt -o ../endpoints/endpoints.txt`
    - `{ cat ../endpoints/params-urls.txt; grep -E '\?' ../endpoints/endpoints.txt; } | sort -u > ../endpoints/params-urls.txt.tmp && mv ../endpoints/params-urls.txt.tmp ../endpoints/params-urls.txt`
    - Add each useful endpoint or finding to `api-findings.txt`, `client-side-findings.txt`, or `secrets.txt` with its source file and line number.
    - Prioritise undocumented API routes, privileged features, upload and download flows, authentication code, GraphQL and WebSocket endpoints, and non-production hosts that are explicitly in scope.
    - Validate interesting findings safely through normal authorised testing and feed confirmed endpoints back into the main recon process.