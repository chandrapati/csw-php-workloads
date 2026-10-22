# Find PHP workloads with Secure Workload

Thanks to Tim Roth for sharing [Discover and Segment PHP servers with Cisco Secure Workload](https://youtu.be/OSnb8eNuJHE). His description of the video says he identifies servers running PHP, checks whether they are publicly exposed, and builds segmentation policies that limit access and reduce lateral movement while patching is still underway. Auto-generated captions did not download on 22 Oct 2026, so the click path inside the player is not transcribed here.

## Why this works when there are thousands of workloads

Each host agent with package visibility reports the packages it sees. Secure Workload keeps that inventory. An inventory filter runs one query against it and returns the members. You are not scrolling the agent list.

Create the filter once:

1. **Organize > Inventory Filters**.
2. **Create Inventory Filter**.
3. Enter the package query. The (i) button beside Query shows the fields the screen accepts. For this search use Package Info, for example `Package Info = php#8.1.29-1`.
4. If the PHP hosts are spread across scopes, leave **Restrict query to ownership scope** unchecked. The [SaaS user guide](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/3_10/cisco-secure-workload-user-guide-saas-v310/segmentation.html) uses that setting so one filter can see workloads outside a single ownership scope. The guide’s sample query on that page is a CVE. Use the package query here.
5. Save. The member count is the number of workloads that reported the package. That count is the list you hand to the patch team and to the policy.

Repeat the filter for each package string you care about. A customer who wants every PHP interpreter, not one build, adds a filter per version the inventory actually shows, or starts from a host that is known to run PHP, reads the exact `name#version`, and searches for that string. Equality against a version you have already read is the filter to trust. A less-than comparison is documented on this screen. The sort order of PHP version strings was not re-tested on 22 Oct 2026.

The filter stays current. The next agent report that includes the same package adds that host. A host that moves to a fixed package leaves the filter. You do not edit a spreadsheet of a thousand names.

## UI

In an inventory filter, the documented comparison looks like this:

```text
Package Info = php#8.1.29-1
```

Cisco’s 2021 Japan field article shows the same screen with `Package Info = php#7.2` and `Package Info < openssl#1.0.2k`, plus a separate `Package CVE` filter: [Cisco Japan, January 2021](https://gblogs.cisco.com/jp/2021/01/cisco-secure-workload_segmentation/). On the lab tenant the interpreter package was either `php#8.1.29-1.el7.remi` or `php#5.4.16-48.el7`.

Package visibility has to be enabled on the agent config profile. Packages inside a container are not assumed to appear on the host inventory. A host with no package report cannot be found by this query. The empty result means “not reported,” and those agents still need a look.

## Are they publicly exposed?

Tim’s second step is exposure, not just presence. After the filter returns the set:

- Open the flows or conversations for those workloads. The user guide’s policy review starts with critical ports and **internet-facing traffic**, then **View Conversations** and **Flow Search** for a specific policy.
- A PHP host whose recent flows show clients from outside the organization is the one to contain first.
- A PHP host that only talks to its own application tier is still unpatched, and it is a smaller reachability problem.

The video description states that he checks public exposure. The exact screen he clicks for that check was not transcribed. Do not treat “the package is installed” as “the host is on the internet.”

## API

Inventory search on the lab tenant does not expose a package dimension. `GET /openapi/v1/inventory/search/dimensions` on 22 Oct 2026 returned no package field. Do not guess a field name for `Package Info`.

The read that worked is per workload:

```text
GET /openapi/v1/workload/{sensor_uuid}/packages
```

Each row has `name`, `version`, `architecture`, `publisher`, and `cve_ids`. Match `name` exactly. `php` is not `php-fpm` and not `gd3php`. `cve_ids` on the package row can be empty even when `GET /openapi/v1/workload/{uuid}/vulnerabilities` has hundreds of rows.

Use a sensor UUID from `GET /openapi/v1/sensors` where the agent is a host agent and `enable_package_visibility` is true.

When [csw-mcp](https://github.com/chandrapati/csw-mcp) is connected, `find_workloads_by_package` runs that read across the sensors and returns hostname, platform, and version. It does not create a filter or a policy.

## What the lab showed on 22 Oct 2026

58 agents. 27 had a package list. The package named `php` was on 10 hosts:

| Version reported | Hosts |
|---|---|
| `8.1.29-1.el7.remi` | 2 web hosts |
| `5.4.16-48.el7` | 8 hosts |

The vulnerability list on one `8.1.29` web host had 506 records. CVE-2026-109618, CVE-2026-109619, and CVE-2024-4577 were not among them. Absence from that list is not a statement that the code is safe. It means this product feed had not mapped those ids on that host that day.

## How to read the version

| Package you find | What it is |
|---|---|
| `php` `8.1.29-*` | June 2024 PHP security release. Fixes from that release include CVE-2024-4577, CVE-2024-5458, and CVE-2024-5585. [PHP 8.1.29](https://www.php.net/releases/8_1_29.php). |
| `php` `5.4.16-*` | A CentOS 7-era build. PHP 5.4 is outside the branches the PHP project still updates. |
| `8.5.12`, `8.4.27`, `8.3.36`, `8.2.35` | The upstream fixes for CVE-2026-109618 and CVE-2026-109619. [Advisory, 21 Oct 2026](https://www.php.net/archive/2026.php#2026-10-21-5). |

PHP 8.1 needs a vendor backport or an upgrade onto a fixed branch. The advisory says some distributions and LTS vendors received patches in advance, and that the volunteer Remi RPM repo would not have the updates on the day of the advisory.
