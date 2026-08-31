# The Complete Guide to This Project
### (Written so anyone, even with zero background, can understand it)

This is the third project in a small portfolio series. It assumes the
basics from the first project's guide (Git, GitHub, Terraform, CI/CD)
are already familiar. This one is entirely about **networking** — the
part of cloud computing that decides what can talk to what, and how.

---

## Part 1 — Why networking, and why it's usually skipped

Most beginner cloud projects (including the first two in this series)
focus on *what* runs: a website, an API. They rarely explain *how
traffic actually reaches it* — because AWS quietly handles a lot of
networking by default when you use simple services like S3 or Lambda.

A VPC (Virtual Private Cloud) project forces you to build that
"plumbing" yourself, which is exactly why it's a strong interview topic
— it proves you understand the layer underneath the services, not just
the services themselves.

## Part 2 — The core concepts, explained with an analogy

Think of a VPC like a large office building you own:

- **The VPC itself** is the building — a defined address range
  (`10.0.0.0/16`, meaning every address from `10.0.0.0` to
  `10.0.255.255`) that belongs entirely to you.
- **Subnets** are floors inside that building, each with its own smaller
  range of room numbers (`10.0.1.0/24` for the public subnet,
  `10.0.2.0/24` for the private one).
- **The public subnet** is a floor with a door directly to the street
  (the internet) — anyone can walk up and knock.
- **The private subnet** is a floor with no street-facing door at all —
  you can only reach it from inside the building.
- **The Internet Gateway** is the building's actual front door,
  connecting it to the outside world at all — without it, *no* floor,
  not even the public one, could reach the internet.
- **The route table** is the building directory telling traffic "if
  you're headed anywhere outside this building, go out through the
  front door" — a subnet only becomes "public" once its floor's
  directory actually points to that door. Just having a street-facing
  wall isn't enough; the directory has to say to use it.
- **The security group** is a bouncer standing at a specific office
  door, checking exactly which visitors (which ports, which traffic
  types) are allowed in.

## Part 3 — The Terraform files, explained

Same overall pattern as the previous two projects
(`providers.tf`, `variables.tf`, `main.tf`, `outputs.tf`, `backend.tf`).
The new concepts are entirely inside `variables.tf` and `main.tf`.

### New variables: CIDR blocks

```hcl
variable "vpc_cidr"             { default = "10.0.0.0/16" }
variable "public_subnet_cidr"   { default = "10.0.1.0/24" }
variable "private_subnet_cidr"  { default = "10.0.2.0/24" }
```

A "CIDR block" is a compact way of writing an IP address range. The
number after the slash (`/16`, `/24`) says how many addresses are in
that range — a smaller number after the slash means a *bigger* range.
`/16` covers 65,536 addresses (the whole VPC); `/24` covers 256 (one
subnet) — a common, simple pattern for small projects.

### `main.tf`, piece by piece

**The VPC:**
```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true
}
```
Creates the private network itself. The two `enable_dns_*` settings
allow resources inside it to be reached by hostname, not just raw IP —
useful default to turn on.

**The two subnets:**
```hcl
resource "aws_subnet" "public" {
  cidr_block              = var.public_subnet_cidr
  map_public_ip_on_launch = true
  ...
}
resource "aws_subnet" "private" {
  cidr_block = var.private_subnet_cidr
  ...
}
```
Notice: `map_public_ip_on_launch = true` only appears on the public one
— anything launched there automatically gets a public IP address. The
private subnet has no such setting.

**The Internet Gateway:**
```hcl
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
}
```
The literal "front door" of the whole VPC. On its own, it doesn't make
anything public yet — it just makes the door *exist*.

**The route table — the actual thing that makes a subnet "public":**
```hcl
resource "aws_route_table" "public" {
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
}
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}
```
This is the most commonly misunderstood part: a subnet isn't "public"
just because it *could* have public IPs. It's public because its route
table says "any traffic going anywhere (`0.0.0.0/0`) should go out
through the Internet Gateway." The private subnet was never associated
with this route table, so it has no path out — genuinely isolated, not
just "hidden."

**The security group:**
```hcl
resource "aws_security_group" "web" {
  ingress { from_port = 22 ... }  # SSH
  ingress { from_port = 80 ... }  # HTTP
  egress  { ... }                  # all outbound allowed
}
```
`ingress` = incoming traffic rules; `egress` = outgoing traffic rules.
This only allows SSH (remote login) and HTTP (web) traffic in, from
anywhere (`0.0.0.0/0`). In a real production setup, SSH would be
restricted to a specific trusted IP address rather than "anywhere" —
noted as a real next step, not done here to keep the project simple.

**Finding the server image automatically:**
```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}
```
Instead of hardcoding a specific AMI (Amazon Machine Image — basically,
a snapshot of an operating system) ID, which changes over time and
differs by region, this automatically finds whatever the latest official
Amazon Linux 2023 image currently is.

**The actual server:**
```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public.id
  ...
}
```
Placed in the public subnet, using the security group defined above.

## Part 4 — The free-tier instance type problem (a real "guides go stale" lesson)

The first attempt used `instance_type = "t2.micro"` — the instance type
almost every AWS tutorial online still uses, since it was the free-tier
default for over a decade. It failed with:

```
Error: InvalidParameterCombination: The specified instance type is
not eligible for Free Tier.
```

**Why:** AWS changed its Free Tier program for accounts created on or
after July 15, 2025. The old model gave 750 hours/month of `t2.micro`
free for 12 months, guaranteed. Newer accounts instead get a pool of
sign-up credits (up to $100 + $100 more) usable during a limited window,
and the free-tier *eligible instance types* changed too — to `t3.micro`,
`t3.small`, `t4g.micro`, and `t4g.small`. `t2.micro` simply isn't on that
list anymore for new accounts.

**The lesson:** cloud platforms change their rules over time, and a
guide/tutorial being popular doesn't mean it's current. Always check a
service's *current* documentation for anything account-creation-date
dependent, rather than trusting an older walkthrough by default — this
is a genuinely common real-world gotcha, not just a beginner mistake.

## Part 5 — Why this project gets destroyed and redeployed on demand

Because of that same account-type change, this project's AWS resources
aren't left running continuously. A `t3.micro` running 24/7 draws down
the account's limited-time credit pool rather than being "free forever
within limits" the way the older model worked. Rather than risk
that, a second pipeline file was added:

```yaml
# destroy.yml
on:
  workflow_dispatch:
```

`workflow_dispatch` means "only run when someone manually clicks the
button in GitHub" — unlike the main pipeline, which runs automatically
on every push. This gives full control: deploy it when you want to
verify it or show it in an interview, destroy it right after, redeploy
it again anytime by re-running the pipeline or pushing a small change.
The underlying Terraform code never changes — only whether it's
currently applied to real AWS resources or not.

## Part 6 — Command/concept glossary (new items vs the first two projects)

| Term | Plain-language meaning |
|---|---|
| VPC | Your own private, isolated network space inside AWS |
| Subnet | A smaller network segment carved out of a VPC's address range |
| CIDR block | A compact notation for an IP address range (e.g. `10.0.0.0/16`) |
| Internet Gateway | The VPC's connection point to the public internet |
| Route table | The rules deciding where different traffic is allowed to go |
| Security group | A firewall attached to a specific resource (like an EC2 instance) |
| Ingress / Egress | Incoming traffic rules / outgoing traffic rules |
| AMI | A snapshot image of an operating system, used to launch an EC2 instance |
| `workflow_dispatch` | A GitHub Actions trigger that only runs when manually clicked, not automatically |

## Part 7 — How to explain this project in an interview

> "I built a VPC from scratch with Terraform — public and private
> subnets, an internet gateway, route tables, and a security group —
> and deployed an EC2 instance into the public subnet, entirely through
> a CI/CD pipeline with manual approval. I deliberately left out a NAT
> Gateway for the private subnet since it wasn't needed and isn't free,
> which forced me to actually understand what makes a subnet 'public'
> versus 'private' — it's the route table, not just having a public IP
> available. I also added a separate, manually triggered pipeline to
> destroy and redeploy the infrastructure on demand, since I wanted
> control over when real AWS resources (and free-tier credit) were
> actually running."

That story shows real understanding of networking fundamentals and cost
awareness — both things interviewers specifically probe for.

---

*This document, together with the repo's README.md, covers everything
needed to fully understand, explain, and rebuild this project.*
