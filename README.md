# README — Annotator CloudFormation Stack (EC2 + VPC + Docker + CloudWatch)

## What this creates

* **Networking**

  * 1 VPC
  * 2 public subnets across two AZs
  * Internet Gateway + public route table (0.0.0.0/0)

* **Security**

  * **Security Group** `allow-all-sg` (all inbound/outbound 0.0.0.0/0) — tighten for production
  * **EC2 Key Pair** from your provided public key

* **Compute**

  * 1 EC2 instance (default `t4g.small`, ARM64) in PublicSubnet1
  * AMI mapping provided for **us-east-2**: `arm64: ami-0ae6f07ad3a8ef182`
  * Instance Profile + IAM Role with:

    * CloudWatch Logs write permissions

* **Observability**

  * CloudWatch **Log Group** `/docker/mqm`
  * **Metric Filter** scanning logs for `ERROR|Error|error` → `AppLogs/ErrorCount`
  * **Alarms**

    * `docker-log-errors` (on `ErrorCount>=1` over 5 min)
    * `ec2-status-check-failed` (EC2 status checks)

  * **SNS Topic** `docker-log-alerts` (+ optional email subscription)

---

## Parameters (inputs)

* `PublicKeyMaterial` (required): contents of your `~/.ssh/id_ed25519.pub` (or other)
* `KeyPairName` (default `mqm-scorecard-key`)
* `InstanceType` (t4g.\* only)
* `AvailabilityZone1` (default `us-east-2a`)
* `AvailabilityZone2` (default `us-east-2b`)
* `AlarmEmail` (optional): email to subscribe to alert topic

---

## Prereqs

* AWS CLI configured with credentials/region.
* Use **us-east-2** (AMI map only provided for that region).
* Your SSH public key on hand.

---

## Deploy

Paste entire `.yaml` file into the cloudformation create stack form.

If you set `AlarmEmail`, **confirm** the SNS subscription email before expecting notifications.

---

## What runs on the instance

Cloud-init installs Docker and configures the Docker **daemon** to ship container logs to CloudWatch Logs:

* Log driver: `awslogs`
* Log group: `/docker/mqm`
* Region: `us-east-2`
* Docker is enabled and restarted
* The ubuntu user is added to the `docker` group

> The user data assumes an **Ubuntu**-based ARM64 AMI (apt, `ubuntu` user).

---

## Connect

1. Get outputs:

   ```bash
   aws cloudformation describe-stacks --stack-name "$STACK" --region "$REGION" \
     --query "Stacks[0].Outputs"
   ```

   Note `PublicIP`, `InstanceId`, `DockerLogGroupName`, `LogErrorAlarmName`.

2. SSH (default user `ubuntu` for default ami):

   ```bash
   ssh -i ~/.ssh/id_ed25519 ubuntu@<PublicIP>
   ```

---

## Logs & Alerts

* **Container logs** (from any containers started with default daemon settings) go to CloudWatch Logs group `/docker/mqm`.
* **Metric filter** increments `AppLogs/ErrorCount` when log lines include `ERROR`, `Error`, or `error`.
* **Alarms**

  * `docker-log-errors`: fires on first error in a 5-min period (treats missing data as OK).
  * `ec2-status-check-failed`: monitors EC2 status checks every 60s.

---

## Typical next steps

* Run your container(s):

  ```bash
  sudo docker run --name app -d <your-image>
  ```

  No extra `--log-driver` needed; daemon.json already sets `awslogs`.
* Verify logs in CloudWatch → Log groups → `/docker/mqm`.

---

## Security notes

* **Ingress is wide open (all ports, all IPs).** For production, restrict:

  * Limit inbound to SSH only (port 22) and/or app ports, and limit source CIDRs.
  * Consider SSM Session Manager instead of SSH.
* Key pair is created from your provided public key; protect the private key.

---

## Cost

* EC2 instance (t4g.\*) + EBS + data transfer
* CloudWatch Logs ingestion/storage + Alarms + SNS
* VPC/IGW have no direct hourly cost but data egress does

---

## Update / Delete

* Update parameters and re-deploy with `aws cloudformation deploy`.
* Delete:

  ```bash
  aws cloudformation delete-stack --stack-name "$STACK" --region "$REGION"
  ```

  Wait for stack deletion to complete; IP/DNS will be released.

---

## Outputs (for scripting)

* `InstanceId`
* `PublicIP`
* `InstanceARN`
* `LogErrorAlarmName`
* `DockerLogGroupName`

---

## Troubleshooting

* **No SSH:** Security Group may need port 22 open from your IP; instance may not be Ubuntu/ARM64; check system status checks.
* **No logs:** Ensure containers are running; verify `/etc/docker/daemon.json` and restart Docker; check IAM role permissions.
* **No alerts:** Confirm SNS email subscription; verify that log lines actually contain `ERROR|Error|error`.
