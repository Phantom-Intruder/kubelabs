# New Relic Proxy

For this lab, we will be setting up a **A Squid proxy server** on an Amazon Linux 2 EC2 instance (public subnet, public IP). * **Private subnet instances** (APM agents, Kubernetes nodes) forward New Relic data via HTTP proxy (to this Squid proxy). We also Configure **Security Groups**, **routing**, and **proxy settings** configured to enable traffic without going through NAT Gateway. The final step is to confirm with New Relic metrics arriving and Squid logging proxy connections.

## Steps

For starters, run a small ec2 instance in a public subnet. Even a small machine would be able to handle quite a lot of data so there is no reason to run anything beyond that considering that the machine will be running forever.
---

### 2. **Install and Configure Squid Proxy**

**Commands:**

```bash
sudo yum install squid -y
sudo systemctl enable squid
sudo systemctl start squid
```

**Key squid.conf changes:**

* Set `http_port 3128`
* Allow traffic from all your 192.168.x.x subnets via:

  ```bash
  acl localnet src 192.168.0.0/16
  http_access allow localnet
  ```
* Keep `http_access deny all` at the end
* Optional: enable access logging (`/var/log/squid/access.log`)

---

### 3. **Security Group Configuration**

**Squid EC2 SG:**

* Inbound:

  * TCP 3128 from **192.168.0.0/16** (all private subnets)
* Outbound:

  * Allow all (default)

**Client SG (K8s nodes / apps):**

* No change needed if using default outbound rules

---

### 4. **Verify Proxy Works**

From a **private subnet instance**:

```bash
curl -x http://<squid-private-ip>:3128 https://collector.newrelic.com/status/mongrel
```

Should return: `{}`
If not, check SGs and subnet route tables.

---

### 5. **Update New Relic Agent Configs to Use Proxy**

In **Java APM Agent YAML**:

```yaml
proxy_host: <squid-private-ip>
proxy_port: 3128
```

In Kubernetes metrics integrations:

* If using New Relic Infrastructure Agent or K8s integration:

  * Set proxy env vars:

    ```yaml
    HTTP_PROXY=http://<squid-private-ip>:3128
    HTTPS_PROXY=http://<squid-private-ip>:3128
    ```

---

### 6. **Test and Validate**

* Confirm metrics reach New Relic dashboards.
* Tail Squid logs:

  ```bash
  sudo tail -f /var/log/squid/access.log
  ```
* Look for lines like:

  ```
  TCP_TUNNEL/200 CONNECT collector.newrelic.com:443
  ```

---

### 7. **Understand Traffic Path**

* App in private subnet → proxy in public subnet via **private IP**
* Proxy EC2 → internet via **IGW**, **bypassing NAT Gateway**
* No traffic crosses NAT Gateway anymore

---

### 8. **Cost Model Summary**

| Method          | Cost per GB | Monthly Infra | Total Monthly (for 5.7TB) |
| --------------- | ----------- | ------------- | ------------------------- |
| NAT Gateway     | \$0.045     | \$32.40       | \~\$295                   |
| EC2 + Squid/IGW | \$0.09      | \~\$16 EC2    | \~\$532                   |

✅ In your case, **NAT is actually cheaper**.
But if data volume drops or if you want central proxying for other reasons (control, audit, IP allowlists), EC2 proxy is still valuable.

---

## 🔁 Extras You Did

* Used **SSM Session Manager** to connect (ensuring proper IAM role and SSM agent setup)
* Assigned public IP via EC2 console
* Verified Squid access with `curl`
* Identified and fixed SG misconfig (initial 403s and timeouts)
* Switched from testing with public IP to private IP for proxy access
* Observed real traffic in Squid logs after config changes

---

Let me know if you’d like to automate any part of this setup (e.g., via Terraform or a Launch Template), or monitor the proxy usage over time.
