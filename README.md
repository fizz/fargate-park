# fargate-park

Park an ECS/Fargate environment you are not using, and bring it back exactly as it was.

Fargate bills for the task reservation, not for work done, so an idle service at `desiredCount=1` costs what a busy one costs. The NAT gateway and the load balancer in front of it keep billing too. `fargate-park` scales the services to zero and, if you ask, deletes the NAT gateway and the load balancer. It records everything it changes in the target account's SSM Parameter Store, so `unpark` needs no flags and no memory of how the park was done.

## Usage

```bash
fargate-park status --cluster my-cluster --profile my-profile --region us-east-1

fargate-park park   --cluster my-cluster --profile my-profile --region us-east-1 --dry-run
fargate-park park   --cluster my-cluster --profile my-profile --region us-east-1 --deep --dns-profile dns-account

fargate-park unpark --cluster my-cluster --profile my-profile --region us-east-1
```

`--cluster`, `--profile` and `--region` are always required. A stale `AWS_REGION` in the shell sends calls to the wrong endpoint and fails as `InvalidClientTokenId`, which looks like an expired session, so the tool never inherits them.

| Flag | Effect |
|---|---|
| `--only a,b` | Park only these services. |
| `--with-nat` | Delete the NAT gateways too. |
| `--release-eip` | Also release the NAT gateways' Elastic IPs. |
| `--with-alb` | Delete the load balancer too, after recording it. |
| `--deep` | `--with-nat --release-eip --with-alb` together: a full weekend park. |
| `--dns-profile A,B` | Profiles whose Route53 zones hold records aimed at the load balancer. |
| `--no-dns` | The load balancer has no public name. Park without DNS records. |
| `--alb-adopt` | Terraform manages the load balancer. Unpark builds nothing and adds back only what Terraform does not own. |
| `--alb-only` | Park or unpark only the load balancer. Services and NAT stay as they are. |
| `--dry-run` | Print every change and make none. |

`unpark` reads its choices from the record. You do not repeat the park flags.

## What it does

### Services

Park records each service's `desiredCount` in `/fargate-park/<cluster>`, then sets it to 0. Unpark sets each service back to the recorded count. A second park of a parked cluster sees every service at 0, so it merges onto the record instead of overwriting it. The counts you need for unpark survive.

### NAT gateway (`--with-nat`)

A NAT gateway costs about $32 a month with nothing behind it. Park deletes it and records its subnet and the routes that pointed at it. Unpark creates a new one and points the routes at it.

The Elastic IP stays by default. An idle address costs about $3.60 a month and keeps the public IP stable, so allowlists downstream still work after unpark. `--release-eip` gives up the address. Unpark then allocates a new one, which is a different IP. Do not use it where a customer allowlists the address.

Unpark brings the NAT gateway up before any service scales. Fargate tasks in private subnets reach ECR and CloudWatch Logs through it. Without it the tasks cannot pull images, and the symptom is a crash loop, not a missing route.

### Load balancer (`--with-alb`)

An idle Application Load Balancer costs about $16 to $22 a month and cannot scale to zero, so park deletes it and unpark builds it again from the record.

The record holds what dies with the load balancer: its attributes, listeners, listener rules, deletion protection, and the WAF web ACL associated with it. Target groups, security groups, subnets and certificates survive the delete and are not recorded. The record goes in its own parameter, `/fargate-park/<cluster>-alb`, because a Standard SSM parameter holds 4096 bytes and the rules use most of that.

### DNS

A new load balancer gets a new DNS name. A record left pointing at the old name points at a hostname the account no longer owns: a dangling record, and a subdomain takeover risk. So park finds every Route53 record aimed at the load balancer, records it and deletes it. Unpark creates each record again against the new name.

The zone often lives in a different account from the load balancer, for example when `example.com` delegates `app.example.com` to a DNS account. Name those profiles with `--dns-profile`. The list goes into the record, so unpark finds the zones without the flag.

If a `--with-alb` park finds no record aimed at the load balancer, it stops before it changes anything. Pass `--no-dns` when the load balancer really has no public name.

### Terraform-managed load balancers (`--alb-adopt`)

If Terraform built the load balancer, a load balancer that unpark builds has a new ARN, and the next `terraform apply` builds a second one. With `--alb-adopt`, park deletes the load balancer as usual, and unpark builds nothing. It requires the load balancer and its listeners to exist already, from your `terraform apply`, and adds back only the listener rules your deploy workflows wrote and any DNS record not yet aimed at it.

## Requirements

`bash`, `aws` (CLI v2) and `python3`. Nothing else.

## Permissions

`status` and every `--dry-run` need only the read actions. The tool uses the profile's credentials in the cluster's account, and each `--dns-profile` for Route53.

| Service | Read | Write |
|---|---|---|
| ECS | `ecs:ListServices`, `ecs:DescribeServices` | `ecs:UpdateService` |
| SSM | `ssm:GetParameter` | `ssm:PutParameter`, `ssm:DeleteParameter` |
| EC2 (`--with-nat`) | `ec2:DescribeNatGateways`, `ec2:DescribeRouteTables`, `ec2:DescribeSubnets`, `ec2:DescribeAddresses` | `ec2:DeleteNatGateway`, `ec2:CreateNatGateway`, `ec2:CreateRoute`, `ec2:ReplaceRoute`, `ec2:AllocateAddress`, `ec2:ReleaseAddress` |
| Elastic Load Balancing (`--with-alb`) | `elasticloadbalancing:DescribeLoadBalancers`, `DescribeLoadBalancerAttributes`, `DescribeListeners`, `DescribeRules`, `DescribeTags`, `DescribeTargetGroups` | `elasticloadbalancing:CreateLoadBalancer`, `DeleteLoadBalancer`, `ModifyLoadBalancerAttributes`, `CreateListener`, `CreateRule`, `AddTags` |
| WAF (`--with-alb`) | `wafv2:GetWebACLForResource` | `wafv2:AssociateWebACL` |
| Route53 (each `--dns-profile`) | `route53:ListHostedZones`, `route53:ListResourceRecordSets` | `route53:ChangeResourceRecordSets` |

## Related

- [eks-park](https://github.com/fizz/eks-park): the same idea for EKS node groups.
- [terraform-aws-parked-page](https://github.com/fizz/terraform-aws-parked-page): a maintenance page to point DNS at while an environment is parked.

## License

MIT
