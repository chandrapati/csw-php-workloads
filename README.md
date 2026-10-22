# Find PHP workloads with Secure Workload, and contain them until they are patched

Field reference for customers and Cisco field. This is not an official Cisco document and not an attestation.

Thanks to **Tim Roth** (YouTube: [Timothy Roth](https://www.youtube.com/@TimothyRoth)) for sharing [Discover and Segment PHP servers with Cisco Secure Workload](https://youtu.be/OSnb8eNuJHE). That video is the source for finding PHP servers and segmenting them. The lab counts and the API notes below are separate work.

Two jobs, in this order:

1. **Find them** with Cisco Secure Workload package inventory.
2. **Contain them until the patch** by allowing only the paths the application needs.

The PHP project says there are no complete mitigations. **Updating PHP is the only safe choice.** Segmentation reduces who can open a connection. It does not patch PHP, and it does not stop a request that still arrives through an allowed front door.

## Find them

Secure Workload can group hosts by the installed package the agent reported.

| Where | What to use |
|---|---|
| Inventory filter in the UI | `Package Info = php#8.1.29-1` |
| API, one workload | `GET /openapi/v1/workload/{uuid}/packages` |
| Chat, when [csw-mcp](https://github.com/chandrapati/csw-mcp) is connected | `find_workloads_by_package` with name `php` |

Match the package name **php** exactly. `php-fpm`, `php-cli`, and `gd3php` are different packages. Package visibility has to be on in the agent config profile. An empty package list means that agent has not reported packages.

On a Secure Workload SaaS lab, 22 Oct 2026, that search returned **10 hosts** with the package named `php`:

- **2** at `8.1.29-1.el7.remi`
- **8** at `5.4.16-48.el7`

`8.1.29` is the June 2024 PHP security release. It is the version in Tim Roth’s filter example. The 21 Oct 2026 fixes for CVE-2026-109618 and CVE-2026-109619 are **8.5.12, 8.4.27, 8.3.36, and 8.2.35**. Those two CVE ids were not in the vulnerability list of the `8.1.29` web host that was checked. A reported package is inventory. It is not proof the host is exploitable, and it is not proof the CVE feed has mapped the CVE.

Details: [docs/find-with-csw.md](docs/find-with-csw.md).

## Contain them until the patch

While the PHP update is scheduled, put the hosts from that filter behind an allow-list:

- Allow the approved front door to TCP 443 on the PHP web tier.
- Allow the PHP web tier to its own database only.
- Allow an admin jump host to management access.
- Deny other inbound connections to the PHP web ports.
- Set that workspace catch-all to Deny only after those allows exist, and only in a lab first.

Sample policies: [examples/sample-policies.json](examples/sample-policies.json). They were **not** applied to a cluster.

What this changes before the patch: an unapproved client cannot open a new connection to the PHP port when the agent is enforcing and the policy is published.

What this leaves in place: a client that is still allowed, including the approved front door, can still send the request that reaches the bug. Local process execution on the host is outside this policy. PHP configuration knobs in the advisory reduce specific attack paths and do not make the install safe.

Details: [docs/contain-until-patched.md](docs/contain-until-patched.md).

## Then confirm the update

After the package upgrade or a confirmed vendor backport, refresh inventory and search again. The workload is remediated when the running package is a fixed build, not when the policy was published.

Fixed upstream lines, from the [PHP advisory of 21 Oct 2026](https://www.php.net/archive/2026.php#2026-10-21-5):

| Branch | Fixed release |
|---|---|
| 8.5 | 8.5.12 |
| 8.4 | 8.4.27 |
| 8.3 | 8.3.36 |
| 8.2 | 8.2.35 |

PHP 8.1 and older no longer receive updates from the PHP project. A vendor may backport. The project also published minimal source backports for 7.4, 8.0, and 8.1. The advisory says the volunteer Remi RPM repo would not have the updates on the day of the advisory. A package string ending in `.el7.remi` still has to be checked against the vendor, not assumed fixed because the host was found.

## Claims

Every statement that matters is in [docs/claims.md](docs/claims.md), with a source, what was observed, and what is still a limit.

## License

[Apache-2.0](LICENSE). Copyright 2026 Sujit Chandrapati.
