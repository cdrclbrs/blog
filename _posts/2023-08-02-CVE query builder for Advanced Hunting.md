---
layout: post
title: "Get easily Cert FR CVEs for Advanced Tracking in Defender"
excerpt: "Query builder for tracking Cert FR CVEs in Defender Advanced Tracking"
excerpt_separator: "<!--more-->"
categories:
  - security
tags:
  - MicrosoftDefender
  - Security
last_modified_at: 2023-07-28T11:12:23-02:00
---


I build a quick script to gather CVEs from any webpage and build your own Advanced Hunting kusto query in Defender: 


# Script


>You can download the script here

```python
import requests
import re
import argparse

def extract_cves(url):
   
    response = requests.get(url)
    if response.status_code != 200:
        raise Exception(f"Failed to get the content. HTTP Status: {response.status_code}")
    
    content = response.text
    cve_pattern = re.compile(r'CVE-\d{4}-\d{4,7}')
    cves = cve_pattern.findall(content)
    cves = list(set(cves))
    
    return cves

def main():
    parser = argparse.ArgumentParser(description="Extract CVEs.")
    parser.add_argument("-url", required=True, help="URL to extract CVEs from")

    args = parser.parse_args()

    cve_list = extract_cves(args.url)
    condition_string = " or ".join([f'CveId == "{cve}"' for cve in cve_list])

    # Kusto query Advanced Hunting (adapt what you need)
    query = f"""
DeviceTvmSoftwareVulnerabilities
| where {condition_string}
| project DeviceName, OSPlatform, OSVersion, SoftwareVendor, SoftwareName, CveId, VulnerabilitySeverityLevel, RecommendedSecurityUpdate
"""

    print(query)

if __name__ == "__main__":
    main()

```

Usage:

```python
python script_name.py -url <URL>

```

That way you can easily get the CVE from any Webpage and build your own query on Defender


