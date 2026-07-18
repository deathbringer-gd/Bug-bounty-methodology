# Wide subdomain enumeration

### Key Concepts

The goal of wide subdomain enumeration is the discover as many attack surfaces as possible, esspecially hidden subdomains. We are looking for exposed development
subdomains, exposed staging subdomains and other relevant subdomains that we can test. The goal is to have as many options as possible.

---

### Steps for enumerating subdomains and their attack surface
This will focus on enumerating as much attack surface as possible.

1. Run `subfinder` to find a majority of subdomains. 
    Commands:
    - `subfinder -d target.com -all > domains.txt`

2. Next run`dnsx` to bruteforce any more domains through dns bruteforcing. This step is required as some domains may not be found by subfinder if they are not in Certificate transparency logs. 
    Commands:
    - `dnsx -d target.com -w "/home/g/Hacking/hacking-map/Bug bounty methodology/Recon/namelist.txt" -asn -cdn -all > dns_bruteforce_domains.txt`
3. After this, go to the website https://subdomainfinder.c99.nl/ and download the list of subdomains for your target.
4. Next is to enumerate as many vhosts (virtual hosts) as possible
    Commands:
    - `ffuf -u target.com -H "Host: FUZZ.example.com -w /home/g/Hacking/hacking-map/Bug bounty methodology/Recon/subdomains-top1million-110000.txt"` 
    - Run for a little bit and then filter by the size of the requests with `-fs size` to get valid ones then append ` > ffuf_vhosts.txt` to main command.

5. Use google and other search engines operators to find relevant information on the company. Save any relevant info to the file `dorks.txt`.
    Tools:
    -  https://pentest-tools.com/information-gathering/google-hacking for premade google dorks.
    - Refer to https://ahrefs.com/blog/google-advanced-search-operators/ for any other relevant google search operators.


6. Run `httpx` on all current subdomains to discover which ones are live and to get some other useful information about them. 
    Commands:
    - `httpx -l domains.txt -sc -ip -tech-detect > live_with_info.txt`
    - `httpx -l domains.txt > live.txt`

7. Next run `katana` to gather as many endpoints as possible on subdomains.
    Commands:
    - `katana -list live.txt -d 3 -kf all -H X-Bug-Bounty:HackerOne-ddddddawd -fs rdn > endpoints.txt`
    - Use `katana -list live.txt -d 3 -kf all -headless > endpoints.txt` if pages dont render until javascript does.

8. Run nmap on all subdomains to scan for vulnerable services.
    Commands: 
    - `nmap -iL live.txt -sV --open -sC -T4 > nmap_scan.txt`

9. Enumerate as many parameters as possible.
    Commands:
    - `grep "=" endpoints.txt > params.txt`
    - `gau target.com | grep "=" > params.txt`

10. Do automatic vulnerability testing, looking for low hanging fruit bugs. Check for all 
    Commands:
    - `sif -f live.txt -dnslist small -dork -git -js -cms -headers -sh -cms -c3 -st -sql -lfi -jwt -openapi -cors -redirect -xss -framework -md subdomains_scan.md -H X-Bug-Bounty:HackerOne-ddddddawd`
    - `subzy run --targets live.txt > subdomain_takeover_scan.txt`
    - `gau --subs target.com | gf vulnerability_type > vulnerability_type_domains.txt`
    - `gf vulnerability_type live.txt && gf vulnerabilit_type endpoints.txt`
11. Organise all data to a couple central files. Make sure there is 1 file for params, endpoints, scan results and any other info.
    Commands: 
    - `cat file1.txt file2.txt | sort -u > merged.txt` Merges 2 files and removes dupes.
12. Identify high interest subdomains. Any subdomains with an interesting scan results or name, save to a file of interesting subdomains.

---

### Deep subdomain attack surface enumeration
This will focus on enumerating as many values as possible from individual subdomains instead of dicovery in mass.

1. Gather all information from wide scans of this subdomain into 1 file.
    Commands:
    - `grep "sub.target.com" endpoints.txt`
    - `grep "sub.target.com" params.txt`
    - `grep "sub.target.com" live_with_info.txt`
    - `grep "sub.target.com" ffuf_vhosts.txt`
    - `grep "sub.target.com" dorks.txt`
    - `grep "sub.target.com" nmap_scan.txt`
    - `grep "sub.target.com" subdomains_scan.md`
    - `grep "sub.target.com" subdomain_takeover_scan.txt`
    - `grep "sub.target.com" vulnerability_types_files.txt`

2. Vist the domain and use the wapplyzer extention to discover information about the sites tech stack and save it to the information file.

3. Run `httpx`, `katana` and burpsuite web crawler on the site again to enumerate extra attack surface.
    Commands:
    - `httpx -d target.com -title -tech-detect -status-code -follow-redirects -web-server -ip -cdn -asn`
    - `katana -u target.com -d 8 -headless -js-crawl -jsluice -kf all`
    - Run burpsuite web crawler on the site and save output to a file.

4. Scan the subdomain with `cloud_enum` to enumerate cloud storage.
    Commands:
    - `cd /Hacking/tools/cloud_enum`
    - `uv run python cloud_enum.py -k company_name`
    - `aws s3 ls s3://bucket_name --no-sign-request --recursive` For listing files in exposed aws buckets.
    - `aws s3 cp s3://bucket_name/path --no-sign-request` For listing file content in exposed aws buckets.

5. Do further google dorking for sensitive information.
    Tools:
    - https://pentest-tools.com/information-gathering/google-hacking
    - Refer to https://ahrefs.com/blog/google-advanced-search-operators/ for any other relevant google search operators.

6. Enumerate extra parameters for testing. Can do this on multiple endpoints.
    Commands:
    - `ffuf -u target.com/page?FUZZ=test -w "/home/g/Hacking/hacking-map/Bug bounty methodology/Recon/raft-large-words.txt"`

7. For any api calls, attempt to enumerate more api parameters.
    Commands:
    - `ffuf -u https://target.com/api -X POST -H "Content-Type: application/json" -d '{"FUZZ":"test"}' -w "/home/g/Hacking/hacking-map/Bug bounty methodology/Recon/large.txt"`