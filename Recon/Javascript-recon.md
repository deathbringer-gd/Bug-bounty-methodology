### JavaScript beautifying an reconisance

The aim of javascript analysis is to discover secrets the developers did not intend people to know, yet have unknowingly exposed them in their code.

---
### Steps for javaScript analysis

1. Gather all endpoints with .js in them and gather all source maps you can find that end in `.map`.
    Commands:
    - `grep -E "\.js($|\?)" endpoints.txt > js-files_endpoints.txt`
    - `grep -E "\.map($|\?)" endpoints.txt > source-maps.txt`
    - `curl -s https://target.com | grep -oE 'src="[^"]+\.js[^"]*"' | sed -E 's/[?#].*$//' js-files.txt | sort -u > js-files_endpoints_extra.txt`
    - `grep -Rni 'sourceMappingURL' js-files/` to find a source maps.
    - `cat js-files_endpoints.txt js-files_endpoints_extra.txt | sort -u > js-file_endpoints.txt` merge the results when finished.

2. Download all javascript files you have collected.
    Commands:
    - `wget -i js-files.txt -P js-files`
    - `curl -s https://target.com > js-files/js_file_name`

3. Beautify all javascript files you have collected.
    Commands:
    - `find js-files -type f -name '*.js' -exec sh -c 'js-beautify "$1" -o "$1.tmp" && mv "$1.tmp" "$1"' _ {} \;`

4. 

---

## draft below

### JS beautifying on subdomains:
- Get all found .js files from the endpoints.txt files by doing `grep -E "\.js($|\?)" endpoints.txt > js-files.txt`. 
- Also do `curl -s https://target.com | grep -oE 'src="[^"]+\.js[^"]*"' > js-files.txt` to get more js files.
- beutify specific js files using `curl -S https://target.com/file.js  | js-beautify > beautified.js grep -iE "(api.?key|secret|password|token|credential|auth)" beautified.js` to search for secrets. Use grep to search for other context specific keywords.
- To beautify and download all javascript files, use `wget -i js-files.txt` and then beautify them as you need to.
- Use these searches on all beautified files:
	`grep -RniE "token|jwt|bearer|authorization|oauth|refresh" beautified/`
	`grep -RniE "secret|apikey|api_key|password|private|key" beautified/`
	`grep -RniE "admin|dashboard|manage|internal" beautified/`
	`grep -RniE "debug|test|dev|staging|sandbox" beautified/`
  and any other relevant searches.
- Use linkfinder to find more endpoints by going to `/Hacking/tools/LinkFinder` then doing `source venv/bin/activate` then `python linkfinder.py -i target.com/jsfile.js > linkfinder.txt` and merge results with endpoints.txt