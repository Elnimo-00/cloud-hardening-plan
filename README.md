# cloud hardening plan

hardening an owasp juice shop deployment on aws — private subnets behind an application load balancer, a bastion for admin access, scoped iam, and vpc flow logs, all as cloudformation.

![vpc architecture](docs/images/architecture.png)

**[full report (pdf)](report.pdf)** — 36 pages, twenty findings with remediation and
console evidence. submitted as the group final project for cloud security (enpm665)
at the university of maryland.

the starting point was a deliberately weak template from an earlier assessment: one
ec2 instance in a public subnet, a flat security group open to `0.0.0.0/0` on 22, 80
and 443, no iam role, no monitoring, database on the same box.

## what changed

| | before | after |
| --- | --- | --- |
| exposure | public ip, ports 22/80/3000 open to the world | no public ip; only the alb is internet-facing, on 80 |
| subnets | one public subnet, private one unused | two public, two private, across two azs |
| access | ssh straight to the instance | bastion in the public subnet, ssh onward from there |
| identity | no role, long-lived keys | instance profile with a read-only s3 policy |
| compute | one instance | auto scaling group, min 1 max 3, alb health checks |
| visibility | none | vpc flow logs for the whole vpc into cloudwatch |

## requirements

- an aws account, and a region with at least two availability zones
- an existing ec2 keypair for the bastion
- aws cli or the console to deploy the stack

## usage

```bash
aws cloudformation deploy \
  --template-file cloudformation/juice-shop-hardened-fixed.yaml \
  --stack-name juice-shop-hardened \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides KeyPairName=<your-keypair> TrustedSSHCidr=<your.ip>/32
```

the stack outputs the alb dns name; juice shop answers on port 80 there once the
targets pass their health check. tear it down with `aws cloudformation delete-stack`
— the nat gateway and the alb both bill by the hour.

two templates are in `cloudformation/`:

- `juice-shop-hardened.yaml` — as submitted with the report
- `juice-shop-hardened-fixed.yaml` — the same architecture with the missing pieces
  filled in, and the one the command above deploys

## results

from the testing documented in [the report](report.pdf):

| | |
| --- | --- |
| findings closed in the template | 9 of 20 |
| closed in the environment or by aws defaults | 6 |
| left as recommendations | 4 |
| still open | 1 — unrestricted egress |
| external nmap | only tcp/80 answered; 22 and 3000 closed from outside |
| reachability analyzer | no path from the internet to a private instance |
| alb targets | healthy, http 200, across two azs |

![alb target group health checks](docs/images/alb-health.png)

- [architecture](docs/architecture.md) — subnets, traffic path, evidence
- [findings](docs/findings.md) — all twenty, and what actually happened to each
- [validation](docs/validation.md) — how each control was tested
- [template gaps](docs/template-gaps.md) — where the submitted template and the report disagree

## notes

the submitted template does not contain the routing the design depends on — no route
tables, no nat gateway — so as written it cannot reach the internet to install the
application, and the health checks would never pass. the environment in the
screenshots was completed in the console. the corrected template closes that gap
along with the bastion's open ssh rule, imdsv2, volume encryption and egress rules;
it parses but has not been deployed. everything above is network and infrastructure
hardening only. juice shop's own vulnerabilities are untouched by design, and the
alb still terminates plain http, so nothing here is confidential in transit.
