# Find PHP workloads with Secure Workload

Thanks to Tim Roth for sharing [Discover and Segment PHP servers with Cisco Secure Workload](https://youtu.be/OSnb8eNuJHE). The UI example `Package Info = php#8.1.29-1` is the filter pattern from that work.

## UI

In an inventory filter, the documented comparison looks like this:

```text
Package Info = php#8.1.29-1
```

Cisco’s 2021 Japan field article shows the same screen with `Package Info = php#7.2` and `Package Info < openssl#1.0.2k`, plus a separate `Package CVE` filter: [Cisco Japan, January 2021](https://gblogs.cisco.com/jp/2021/01/cisco-secure-workload_segmentation/).

Build one filter per package string you care about. On the lab tenant the interpreter package was either `php#8.1.29-1.el7.remi` or `php#5.4.16-48.el7`. A less-than comparison is documented for this screen. The sort order of PHP version strings was not re-tested on 22 Oct 2026, so equality against the version you read from a host is the filter to trust.

Package visibility has to be enabled on the agent config profile. Packages inside a container are not assumed to appear on the host inventory.

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
