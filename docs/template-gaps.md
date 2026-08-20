# what the submitted template actually contains

the report describes the environment that was deployed and validated in the aws
console. `juice-shop-hardened.yaml` — the template submitted with it — is a
narrower artifact. the console screenshots in the appendix are real; several of the
controls behind them were applied outside this file, or came from account defaults.

worth stating plainly, because a template that claims *"fully hardened"* in its own
description invites exactly this audit.

## the deploy-blocking one

**there are no route tables and no routes.** an internet gateway is created and
attached, but nothing routes `0.0.0.0/0` to it, and the private subnets have no NAT.
consequences, in order:

1. instances in `PrivateSubnet1/2` have no outbound path.
2. the launch template's userdata — `curl` to nodesource, `yum install`, `git clone`,
   `npm install` — cannot reach anything, so juice shop never installs.
3. the target group health-checks `/` on port 3000, gets nothing, and the asg never
   reports a healthy target.

the report's implementation plan lists "NAT Gateway, Route Tables" under *aws
services used* and describes attaching a NAT to the private route table. that step
is in the prose but not in the file.

## claimed in the report, absent from the template

| control | report says | template has |
| --- | --- | --- |
| bastion ssh | "only allows SSH access from trusted IP addresses" | `CidrIp: 0.0.0.0/0` on port 22 |
| nat gateway | provisioned in the public subnet, attached to the private route table | no `AWS::EC2::NatGateway` |
| route tables | listed under services used | none |
| network acls | "we also set up Network ACLs (NACLs)" | no `AWS::EC2::NetworkAcl` |
| imdsv2 | finding 1, "enforced by default … through modern launch templates" | no `MetadataOptions`. the console does show *IMDSv2: Required* — that comes from the al2023 ami default, not from the template |
| encryption at rest | finding 5, which itself says "on by default in many aws accounts … can be explicitly set" | no `Encrypted: true`. the report is upfront that this leans on the account default |
| restricted egress | finding 17 is listed with an empty remediation | no egress rules, so all three groups allow all outbound |
| guardduty | "enable AWS GuardDuty for the region" | not in template (console-level service, reasonable to set outside iac) |
| alb dns output | "output the DNS of the ALB as the new public access point" | no `Outputs` section |

## smaller things

- the s3 policy grants `s3:GetObject` on `Resource: "*"` — every object in every
  bucket the account can see. least privilege on the action, not on the resource.
- `Mappings.RegionMap` is declared and never referenced; the subnets use `!GetAZs`.
- `ManagedPolicyName`, `LaunchTemplateName` and the alb `Name` are hardcoded, so a
  second stack in the same account fails on the name collision.
- the listener is http/80 only. the report names TLS as future work, which is fair,
  but it does mean the "hardened" path is still plaintext end to end.

## the corrected template

`juice-shop-hardened-fixed.yaml` is the same architecture with those gaps closed:
public and private route tables, an igw route, a nat gateway, `TrustedSSHCidr` as a
required parameter with no default, imdsv2 required, gp3 volumes encrypted, explicit
egress on all three security groups, a scoped bucket arn, log group retention, and
the alb dns as an output.

it parses, and the security groups are arranged so the alb group can declare its
egress inline without a dependency cycle. **it has not been deployed** — no aws
account was available while writing it, and `cfn-lint` was not installed, so treat
it as reviewed-by-reading, not validated. nacls are still absent: stateless rules
are easy to get subtly wrong, and a half-correct nacl is worse than none.
