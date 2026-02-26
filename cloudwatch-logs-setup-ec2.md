# Chatterbox CloudWatch Logs Setup

This guide explains how to configure the **Amazon CloudWatch Agent** to stream logs from the `chatterbox.service` application to **CloudWatch Logs**. It ensures that logs are live and accessible to developers for all EC2 instances launched from the AMI.

---

## **1. Prerequisites**

- Ubuntu EC2 instance
- `chatterbox.service` installed at `/home/ubuntu/conversational-ai/server/main.py`
- IAM role attached to the instance with the following permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

- CloudWatch Agent installed:

```bash
sudo apt update
sudo apt install -y amazon-cloudwatch-agent
```

---

## **2. Configure chatterbox.service to log to file**

Update `/etc/systemd/system/chatterbox.service`:

```ini
[Unit]
Description=Mumbai Chatterbox Main Application
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/conversational-ai/server
ExecStart=/usr/bin/python3 /home/ubuntu/conversational-ai/server/main.py
Restart=always
RestartSec=5

# Redirect stdout and stderr to log file
StandardOutput=append:/home/ubuntu/conversational-ai/server/logs/chatterbox.log
StandardError=append:/home/ubuntu/conversational-ai/server/logs/chatterbox.log

[Install]
WantedBy=multi-user.target
```

Create log folder and file:

```bash
mkdir -p /home/ubuntu/conversational-ai/server/logs
touch /home/ubuntu/conversational-ai/server/logs/chatterbox.log
chown ubuntu:ubuntu /home/ubuntu/conversational-ai/server/logs/chatterbox.log
chmod 644 /home/ubuntu/conversational-ai/server/logs/chatterbox.log
```

Reload systemd and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable chatterbox.service
sudo systemctl restart chatterbox.service
```

Check logs locally:

```bash
tail -f /home/ubuntu/conversational-ai/server/logs/chatterbox.log
```

---

## **3. Configure CloudWatch Agent**

Create CloudWatch Agent config file:

```bash
sudo nano /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

Paste the following:

```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/home/ubuntu/conversational-ai/server/logs/chatterbox.log",
            "log_group_name": "chatterbox-logs",
            "log_stream_name": "{instance_id}",
            "timezone": "UTC"
          }
        ]
      }
    }
  }
}
```

---

### **4. Start and enable CloudWatch Agent**

Apply the config and start the agent:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s
```

Check status:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```

- `status: running` → CloudWatch Agent is live
- Logs are streaming to CloudWatch **log group `chatterbox-logs`**, stream `{instance_id}`.

Enable agent to start on boot:

```bash
sudo systemctl enable amazon-cloudwatch-agent
```

---

## **5. Verification**

1. Run the application and generate logs:

```bash
sudo systemctl restart chatterbox.service
```

2. Tail logs locally:

```bash
tail -f /home/ubuntu/conversational-ai/server/logs/chatterbox.log
```

3. Check CloudWatch:

- Navigate to **CloudWatch → Logs → Log Groups → `chatterbox-logs`**
- Confirm a new log stream `{instance_id}` is created and logs appear.

---

## **6. Notes**

- Each new EC2 instance launched from this AMI will automatically send logs to CloudWatch if:
  - `chatterbox.service` is enabled on boot
  - CloudWatch Agent is installed and enabled
  - IAM role has correct CloudWatch Logs permissions
- Old logs from the source AMI are **not sent**. Only new logs generated on the new instance will appear.

---

**Optional:** Developers can use the AWS CLI to tail logs in real-time:

```bash
aws logs tail /chatterbox/server --follow --region <your-region>
```
