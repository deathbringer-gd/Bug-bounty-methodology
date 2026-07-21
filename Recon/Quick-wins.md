# Quick wins

This is a file of things you can sweep a program for fairly quickly that will give you a quick win. Most of the time these will not work however it is worth trying just in case.

---

### Sensitive-file exposure checks

Run these checks on a host after recording a normal missing-path response. This is a small, high-signal check for accidentally deployed configuration, source-control metadata, backups, and development artifacts; it is not a reason to download databases, repositories, private keys, or customer data.

1. Create a per-host results directory and baseline the site's not-found response. Some applications return a custom `200` page for every unknown path, so compare status, length, title, and response body before treating a result as real.
    Commands:
    - `mkdir -p sensitive-files`
    - `curl -sS -L -D sensitive-files/not-found-headers.txt -o sensitive-files/not-found-body.txt "https://sub.target.com/__sensitive-file-baseline-$(date +%s)"`
    - `wc -c sensitive-files/not-found-body.txt`

2. Begin with read-only `GET` requests and save metadata separately from response bodies.
    Commands:
    - `ffuf -w "/home/g/Hacking/hacking-map/Bug bounty methodology/Recon/wordlists/sensitive-files.txt" -u "https://sub.target.com/FUZZ" -ac -rate 10 -mc 200,206,301,302,307,308 -of json -o sensitive-files/ffuf-results.json`
    - `jq -r '.results[] | [.status, .length, .words, .lines, .url, .redirectlocation] | @tsv' sensitive-files/ffuf-results.json > sensitive-files/candidates.tsv`
    - Review candidates against the baseline and record only confirmed paths in `sensitive-files/confirmed.tsv` with URL, status, content type, size, date, and a redacted description.

3. Validate candidates with the minimum evidence necessary. A readable `.git/HEAD`, a configuration filename with an appropriate content type, or a backup file's metadata is normally enough to establish exposure.
    Commands:
    - `curl -sS -I "https://sub.target.com/.git/HEAD" | tee sensitive-files/git-head-headers.txt`
    - `curl -sS -I "https://sub.target.com/.env" | tee sensitive-files/env-headers.txt`
    - `curl -sS -I "https://sub.target.com/backup.zip" | tee sensitive-files/backup-headers.txt`
    - Do not use repository-reconstruction tools, bulk backup downloads, or secret-validation requests unless the programme explicitly allows them and the extra access is necessary for the report.

4. Use context to make the check more precise instead of expanding the list blindly.
    - Test backup suffixes only for files or routes already observed, for example a confirmed `config.php` may justify checking `config.php.bak`, `config.php.old`, and `config.php~`.
    - Review JavaScript, source maps, robots.txt, sitemap.xml, public repositories, archives, error pages, and deployment fingerprints for application-specific filenames and directories. Add reviewed candidates to `sensitive-files/targeted-paths.txt` and probe them at the same conservative rate.
    - Prioritise `.env` variants, `.git/HEAD`, source maps, CI/deployment files, API specifications, debug endpoints, and archives on explicitly in-scope development or staging hosts.