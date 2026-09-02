# EC2 User Data: Serving Instance Metadata on a Web Page

A handy pattern when learning AWS networking is a bootstrap script that installs a web server and renders the instance's own metadata — hostname, instance ID, type, and public/private IPs — on its home page. Hit the instance in a browser and it tells you who it is and whether you reached it over a public or private path. It's a great way to demonstrate public vs private subnets, load balancer target selection, and NAT behavior.

This article walks through such a script, explains the [Instance Metadata Service (IMDS)](#the-instance-metadata-service-imds) it depends on, upgrades it to the more secure **IMDSv2**, and shows how to attach it as EC2 user data.

## The Script

This runs at first boot on an Amazon Linux instance. It reads instance metadata, decides whether the instance is "public" or "private" based on whether it has a public IP, installs Apache, and writes an info page.

```bash
#!/bin/bash
# userdata.sh — install Apache and serve this instance's metadata

# Gather instance metadata (IMDSv1 — see the IMDSv2 version below)
export HOSTNAME=$(hostname)
export PUBLIC_IPV4=$(curl http://169.254.169.254/latest/meta-data/public-ipv4)
export LOCAL_IPV4=$(curl http://169.254.169.254/latest/meta-data/local-ipv4)
export INSTANCE_ID=$(curl http://169.254.169.254/latest/meta-data/instance-id)
export INSTANCE_TYPE=$(curl http://169.254.169.254/latest/meta-data/instance-type)
export ACCESS_CATEGORY="public"

if [ -z "$PUBLIC_IPV4" ]; then
    ACCESS_CATEGORY="private"
else
    ACCESS_CATEGORY="public"
fi

# Install and start Apache
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

# Render the metadata into the default index page
echo "<h1>Hello. You are accessing a <span style='color: red; font-weight: bold;'>$ACCESS_CATEGORY</span> instance:<br></h1>" >> /var/www/html/index.html
echo "<h3>Instance Profile</h3>" >> /var/www/html/index.html
echo "Hostname:      $HOSTNAME<br>" >> /var/www/html/index.html
echo "Instance ID:   $INSTANCE_ID<br>" >> /var/www/html/index.html
echo "Instance type: $INSTANCE_TYPE<br><br>" >> /var/www/html/index.html
echo "Public IP:  <span style='color: red'>$PUBLIC_IPV4</span><br>" >> /var/www/html/index.html
echo "Private IP: <span style='color: red'>$LOCAL_IPV4</span>" >> /var/www/html/index.html
```

## How It Works, Step by Step

1. **Metadata lookup** — each `curl` to `169.254.169.254` returns one metadata field. `public-ipv4` only exists if the instance actually has a public IP.
2. **Public vs private detection** — `[ -z "$PUBLIC_IPV4" ]` is true when the public IP string is empty, so no public IP means the script labels the instance `private`. An instance in a private subnet (or without a public IP association) falls into this branch.
3. **Web server install** — `yum` installs and enables Apache (`httpd`) so it starts on boot.
4. **Page rendering** — the script appends HTML to Apache's default `index.html`, coloring the access category and IPs so they stand out.

## The Instance Metadata Service (IMDS)

Every EC2 instance can query a link-local address, **`169.254.169.254`**, to learn about itself — no credentials or internet needed. That's the IMDS. Common paths under `/latest/meta-data/`:

| Path | Returns |
|------|---------|
| `instance-id` | The instance's ID (`i-0abc...`) |
| `instance-type` | Instance type (`t3.micro`) |
| `local-ipv4` | Primary private IPv4 |
| `public-ipv4` | Public IPv4 (only if one is assigned) |
| `placement/availability-zone` | AZ the instance runs in |
| `mac` / `network/interfaces/...` | Network interface details |
| `iam/security-credentials/<role>` | Temporary IAM role credentials |

That last one is why IMDS security matters: anything that can reach `169.254.169.254` from inside the instance can read the instance's IAM credentials.

## Upgrade to IMDSv2 (Recommended)

The script above uses **IMDSv1**, a simple request/response model. AWS now defaults new instances to **IMDSv2**, which requires a short-lived session token first. IMDSv2 mitigates SSRF attacks where a tricked application is used to reach the metadata endpoint. On instances configured to require IMDSv2, the plain `curl` calls above return nothing — so use the token flow:

```bash
#!/bin/bash
# userdata.sh — IMDSv2 version

# 1. Get a session token (valid up to 6 hours)
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# 2. Pass the token on every metadata request
imds() {
  curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    "http://169.254.169.254/latest/meta-data/$1"
}

export HOSTNAME=$(hostname)
export PUBLIC_IPV4=$(imds public-ipv4)
export LOCAL_IPV4=$(imds local-ipv4)
export INSTANCE_ID=$(imds instance-id)
export INSTANCE_TYPE=$(imds instance-type)

ACCESS_CATEGORY="public"
[ -z "$PUBLIC_IPV4" ] && ACCESS_CATEGORY="private"

yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

{
  echo "<h1>Hello. You are accessing a <span style='color:red;font-weight:bold;'>$ACCESS_CATEGORY</span> instance:</h1>"
  echo "<h3>Instance Profile</h3>"
  echo "Hostname:      $HOSTNAME<br>"
  echo "Instance ID:   $INSTANCE_ID<br>"
  echo "Instance type: $INSTANCE_TYPE<br><br>"
  echo "Public IP:  <span style='color:red'>${PUBLIC_IPV4:-none}</span><br>"
  echo "Private IP: <span style='color:red'>$LOCAL_IPV4</span>"
} >> /var/www/html/index.html
```

Grouping the `echo` lines in a `{ ... } >> file` block is tidier than repeating the redirect, and `${PUBLIC_IPV4:-none}` prints `none` instead of an empty string for private instances.

## Amazon Linux 2 vs 2023

The `yum` + `httpd` commands work on Amazon Linux 2. Amazon Linux 2023 aliases `yum` to `dnf`, so the script still runs, but the idiomatic form is:

```bash
dnf update -y
dnf install -y httpd
systemctl enable --now httpd
```

For Ubuntu/Debian AMIs, swap in `apt`:

```bash
apt-get update -y
apt-get install -y apache2
systemctl enable --now apache2
```

## Attaching It as User Data

User data runs once at first boot (as root, via cloud-init). Attach it a few ways:

### AWS CLI

```bash
aws ec2 run-instances \
  --image-id ami-0abc123... \
  --instance-type t3.micro \
  --key-name my-key \
  --security-group-ids sg-0abc123... \
  --subnet-id subnet-0abc123... \
  --user-data file://userdata.sh
```

### Terraform

```hcl
resource "aws_instance" "web" {
  ami                    = "ami-0abc123..."
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  user_data              = file("${path.module}/userdata.sh")

  # Force IMDSv2
  metadata_options {
    http_tokens   = "required"
    http_endpoint = "enabled"
  }
}
```

In the console, paste the script under **Advanced details → User data** when launching.

## Security Group Requirement

To view the page you must allow inbound HTTP. For a public instance, open port 80 to your traffic source:

```
Type: HTTP | Protocol: TCP | Port: 80 | Source: 0.0.0.0/0   (or your IP/CIDR)
```

A private instance has no public IP, so you'd reach port 80 from within the VPC, through a bastion, or behind a load balancer — which is exactly the scenario the "public/private" label helps illustrate.

## Verifying and Troubleshooting

```bash
# Watch user-data / cloud-init execution on the instance
sudo cat /var/log/cloud-init-output.log

# Re-run user data manually for testing (Amazon Linux)
sudo cloud-init clean && sudo cloud-init init

# Confirm Apache is up
systemctl status httpd
curl http://localhost
```

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Page loads but IPs are blank | IMDSv2 required, script used IMDSv1 | Use the token-based version above |
| Can't reach the page at all | Security group / no public IP / no route | Open port 80; check public IP and route table |
| `httpd` not found on AL2023 | Package differences | Use `dnf install -y httpd` |
| User data didn't run | Not a first boot, or script error | Check `/var/log/cloud-init-output.log` |

## Why This Is Useful

- **Teaching public vs private subnets** — launch one instance in each and compare the pages.
- **Load balancer demos** — put several behind an ALB and refresh to watch the instance ID change across targets.
- **Quick connectivity checks** — confirms user data, IMDS access, and inbound rules all work end to end.

Treat it as a learning/demo tool. It exposes instance metadata publicly by design, which you would never do on a real production workload.

## References

- [Instance metadata and user data (IMDS)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html) — official AWS docs
- [Use IMDSv2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html) — official AWS docs
- [Run commands at launch with user data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html) — official AWS docs

*Content was rephrased for compliance with licensing restrictions.*
