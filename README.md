# Annotator CloudFormation Stack (EC2 + VPC + Docker + CloudWatch)


## Create CloudFormation Stack

1. Visit [CloudFormation](https://us-east-2.console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/create)
2. Choose an existing template. To specify the template, select `Upload a template file`.
3. Click `Choose file`.
4. Select the file `cloudformation.yaml` from this repository.
5. Click next
6. You should be on a page titled: Specify stack details. There are a few required parameters on this page:
    - Stack Name: enter whatever you would like.
    - [OPTIONAL] Alarm Email: enter an email. This email will receive a confirmation email after creation of
      this stack, requesting confirmation of signup to receive alerts in the case of alarms.
    - Public Key Material: Paste your public key in the field, as generated with ssh-keygen. See any number of
      tutorials online for instruction. Here is a good one by [github](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
7. Click next.
8. Scroll to the bottom and select the checkbox in the blue box labelled: `I acknowledge that AWS CloudFormation might create IAM resources with custom names.`
9. Click next.
10. You should now be on the review page. Click next.

CloudFormation will now proceed to create the stack.

---

## Connect

1. After the stack creation completes, you may connect to the created server. First we must find the
   it's public ip address. Select the stack on the cloudformation page and navigate to the
   `Resources` tab.
2. The Logical ID column of the resource table should contain a row with the contents:
   `EC2Instance`. Click on the corresponding physical ID link.
3. On the ec2 instance dashboard, if no instances are displayed, you may need to clear the filters.
   Click on the instance ID and it will display the Instance summary. Click on the copy button next
   to the public IPv4 address. The address will now be in your system's clipboard.
4. Open a terminal on your system with the `ssh` command available.
   Connect to the server, specifying the path to the private key which corresponds to the public key
   you specified when you created the stack. Use `ubuntu` as the user name and paste the public ip
   address that you copied in the last step.
   ```bash
   # example key path specified after the -i option.
   ssh -i ~/.ssh/id_ed25519 ubuntu@<PublicIP>
   ```
   Then type `yes` when prompted.
5. You should now be connected to the server. It's time to deploy the application.
   ```bash
   git clone https://github.com/byu-trg-team/mqm-scorecard-api
   cd mqm-scorecard-api
   ```
6. Create a `.env` file according to the README file in the current directory.
7. Start the containers:
   ```bash
   docker compose up -d
   ```
8. See the api documentation for instructions on getting started and troubleshooting the
   application. The vast majority of errors that may occur at any point are related to an invalid
   `.env` file. To view the latest errors in any container, use `docker logs`.
   ```bash
   docker ps -a # view all containers, including stopped containers.
   # The previous command displays container ids.
   docker logs <container-id> -n 20 # view the last 20 lines of logs from a given container.
   # View all logs for all containers in the stack:
   docker compose logs
   ```

---

## Sparse Technical Specifications

###### Network and other info is glossed over, see the cloudformation template for more information.

* **Compute**

  * 1 EC2 instance (default `t4g.small`, ARM64) in PublicSubnet1

* **Observability**

  * CloudWatch **Log Group** `/docker/mqm`
  * **Metric Filter** scanning logs for `ERROR|Error|error` → `AppLogs/ErrorCount`
  * **Alarms**

    * `docker-log-errors` (on `ErrorCount>=1` over 5 min)
    * `ec2-status-check-failed` (EC2 status checks)

  * **SNS Topic** `docker-log-alerts` (+ optional email subscription)

---

## Logs & Alerts

* **Container logs** (from any containers started with default daemon settings) go to CloudWatch Logs group `/docker/mqm`.
* **Metric filter** increments `AppLogs/ErrorCount` when log lines include `ERROR`, `Error`, or `error`.
* **Alarms**

  * `docker-log-errors`: fires on first error in a 5-min period (treats missing data as OK).
  * `ec2-status-check-failed`: monitors EC2 status checks every 60s.

---

## Troubleshooting

* **No SSH:** Security Group may need port 22 open from your IP; instance may not be Ubuntu/ARM64; check system status checks.
* **No logs:** Ensure containers are running; verify `/etc/docker/daemon.json` and restart Docker; check IAM role permissions.
* **No alerts:** Confirm SNS email subscription; verify that log lines actually contain `ERROR|Error|error`.
