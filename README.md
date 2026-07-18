### Bug bounty methodology

This is my own comprehensive bug bounty methodology for reconisance and explotation. Currently all of the `Explotation` folder is in first ai drafts. This guide is tailored to my own setup on linux mint however anyone can follow it and setup a similar environment. Refer to the setup guide to setup all the nessarsary tools.

---

### File structure
The file structure should be as follows for each program:
```
target.com/
│
├── recon/
│   │
│   ├── automatic-recon/
│   │   │
│   │   ├── notes.md
│   │   ├── domains.txt
│   │   ├── live.txt
│   │   ├── live_with_info.txt
│   │   ├── ffuf_vhosts.txt
│   │   ├── dorks.txt
│   │   ├── endpoints/
│   │   │  ├── endpoints.txt
│   │   │  └── params.txt
│   │   ├── javascript/
│   │   │  ├── js-files_endpoints.txt
│   │   │  ├── secrets.txt
│   │   │  ├── source-maps.txt
│   │   │  └── js-files/
│   │   └── vulnerability_scans/
│   │       ├── subdomains_scan.md
│   │       ├── nmap_scan.txt
│   │       ├── subdomain_takeover_scan.txt
│   │       └── vulnerabilitys/
│   │           ├── xss.txt
│   │           ├── aws-keys.txt
│   │           ├── cors.txt
│   │           ├── debug-pages.txt
│   │           ├── debug-logic.txt
│   │           ├── firebase.txt
│   │           ├── idor.txt
│   │           ├── http-auth.txt
│   │           ├── img-traversal.txt
│   │           ├── interestingEXT.txt
│   │           ├── interestingparams.txt
│   │           ├── interestingsubs.txt
│   │           ├── lfi.txt
│   │           ├── rce.txt
│   │           ├── redirect.txt
│   │           ├── s3-buckets.txt
│   │           ├── sqli.txt
│   │           ├── ssrf.txt
│   │           ├── ssti.txt
│   │           ├── takeovers.txt
│   │           └── php-sinks.txt
│   │
│   └── manual-recon/
│       │
│       ├── notes.md
│       ├── interesting-endpoints.md
│       └── interesting_subdomain_name/
│           ├── all-endpoints.txt
│           ├── interesting-endpoints.md
│           ├── cloud_storage.md
│           ├── authentication.md
│           ├── authorization.md
│           ├── business-logic.md
│           ├── api-notes.md
│           ├── javascript-review.md
│           └── technology-notes.md
│
├── exploitation/
│   │
│   ├── xss.md
│   ├── sqli.md
│   ├── ssrf.md
│   ├── csrf.md
│   ├── idor.md
│   ├── xxe.md
│   ├── lfi.md
│   ├── rfi.md
│   ├── rce.md
│   ├── ssti.md
│   ├── open-redirect.md
│   ├── clickjacking.md
│   ├── cors.md
│   ├── authentication.md
│   ├── authorization.md
│   ├── file-upload.md
│   ├── path-traversal.md
│   ├── information-disclosure.md
│   ├── business-logic.md
│   ├── race-conditions.md
│   ├── graphql.md
│   ├── deserialization.md
│   ├── prototype-pollution.md
│   ├── request-smuggling.md
│   ├── web-cache-poisoning.md
│   ├── web-cache-deception.md
│   ├── host-header.md
│   ├── jwt.md
│   ├── oauth.md
│   ├── webhooks.md
│   ├── cloud-storage.md
│   └── reports/
│       ├── drafts/
│       └── submitted/
│
├── scope.md
├── out-of-scope.md
├── commands.md
└── README.md
```

---

### Setup

These are all the tools used in this process and how to set them up accordingly.



---

### Useful resources
- https://github.com/danielmiessler/seclists for bruteforcing lists.
- https://portswigger.net/web-security/all-topics for docs on vulnerability information.
- https://hacktricks.wiki/en/pentesting-web/web-vulnerabilities-methodology.html great for web explotation notes and expoitation.
- https://oreobiscuit.gitbook.io/introduction has a huge amount of bug bounty relevant information.
- https://apis.guru/graphql-voyager/ good for graphql schema visualisation.
- https://www.intigriti.com/researchers/blog good for reading about creative ways to exploit vulnerabilitys + docs for learning about niche vulnerability classes.
- https://infosecsanyam.medium.com/web-application-security-bug-bounty-methodology-reconnaissance-vulnerabilities-reporting-635073cddcf2 huge amount of writeups for very complexed vulnerabilitys, worth reading.
- https://www.openbugbounty.org/bugbounty-list/ can be useful for finding niche new programs.
- https://seclists.org/fulldisclosure/ good to find the latest new vulnerabilitys explained well.