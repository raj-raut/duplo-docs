# Deploying Duplo Portal on AWS: Updated Guide

In this guide, we’ll cover how to deploy the Duplo Portal on AWS using a step-by-step process. This includes configuring DNS, certificates, CloudFormation stacks, VPN setup, logging, monitoring, and more. The steps are organized for ease of understanding and practical implementation.

---

## 1. Create a Public Hosted Zone in Route 53
The first step is to set up a public hosted zone for your domain in AWS Route 53.

- **Example DNS**: `prod.customer-dns.com`
- This will be used for certificate creation and CloudFormation stack configuration.

### Steps:
1. Go to AWS Route 53.
2. Create a public hosted zone for your domain (e.g., `prod.customer-dns.com`).

---

## 2. Request Certificates from AWS Certificate Manager (ACM)

We need SSL certificates for secure communication. Two certificates are required: one for the portal and one for logging and monitoring.

- **Certificate 1**: ECDSA P-384 for the portal domain (e.g., `*.prod.customer-dns.com`).
- **Certificate 2**: RSA 2048 for logging and monitoring subdomains (use `*` as a wildcard to cover all subdomains).

### Steps:
1. Go to AWS Certificate Manager.
2. Request an ECDSA P-384 certificate for your domain.
3. Request an RSA 2048 certificate with `*` to cover all subdomains.

**Why use ECDSA and RSA?**  
- **ECDSA P-384**: Provides strong security with smaller key sizes.
- **RSA 2048**: Ensures robust encryption for logging and monitoring services.

---

## 3. Deploy Duplo Portal Using AWS CloudFormation

We will deploy the Duplo Portal by creating a CloudFormation stack.

### Steps:
1. **Prepare the Template**: Use the template from the S3 URL:
   - **S3 URL**: `https://duplo-setup-resources.s3-us-west-2.amazonaws.com/2023.07/duplo-root-stack.json`
   
2. **Specify Stack Parameters**:
   - **Stack Name**: `duplo-root-stack`
   - **Class B**: `220` (for the VPC CIDR, e.g., 10.220.0.0/16)
   - **Setup URL**: e.g., `https://duplo.prod.customer-dns.com`
   - **Master AMI ID**: Use the appropriate AMI-ID for your region from the list below:
     - ap-northeast-1: `ami-0a5a5654af962ef44`
     - ap-south-1: `ami-067e59fd8c1c19857`
     - ap-southeast-2: `ami-0ced1fcaffdf9f867`
     - ca-central-1: `ami-02638e969501cfa82`
     - eu-west-2: `ami-0e58b649241a431a1`
     - us-east-1: `ami-0b66c0e497e55a61b`
     - us-east-2: `ami-03d5f6731f426902e`
     - us-west-1: `ami-0a969ed1c0510538d`
     - us-west-2: `ami-0d37a83cc0eb387a9`

3. **Additional Parameters**:
   - **DUPLODNSPRFX**: e.g., `prod.customer-dns.com`
   - **AWS Extra Regions**: Add additional regions to manage with Duplo, separated by a semicolon.
   - **AWS Route 53 Domain ID**: Enter the Route 53 Hosted Zone ID.
   - **Bastion AMI ID**: `ami-0d4efc14256385b61` (Optional, if you enable Bastion).
   - **Default Admin**: `user_name@duplocloud.net`
   - **Default EBS Encryption**: `YES`
   - **Enable Bastion**: `NO`
   - **Enable ES Role**: `YES`
   - **Enable JIT AWS Admin API**: `true`
   - **Generate Bootstrap Token**: `false`

4. **Create the Stack**: Verify the parameters and click "Create". This will deploy:
   - VPC
   - Load balancer
   - EC2 instance (Duplo-master)

---

## 4. Configure DNS A Record for Load Balancer

Once the CloudFormation stack is deployed, the load balancer DNS needs to be associated with the domain.

### Steps:
1. In Route 53, add an A record for your load balancer’s DNS to the hosted zone.
2. Confirm that the load balancer is accessible using the domain name.

---

## 5. Verify Duplo-Master EC2 Instance is Running

Ensure the Duplo-master instance is operational.

### Steps:
1. Open a browser and navigate to the DNS (e.g., `https://duplo.customer-dns.com`).
2. Verify that Duplo-master is running.

---

## 6. Register Customer and Environment in DevOps Manager

Add your customer and environment in DuploCloud’s DevOps Manager.

### Steps:
1. Navigate to [DevOps Manager](https://devopsmgr.prod-apps.duplocloud.net/).
2. Register your customer.
3. Create a new environment under the customer with the necessary details.

---

## 7. Update Duplo-Master to the Latest Release

Update Duplo-master to the latest_feb_release.

### Steps:
1. Create an AWS profile in `~/.aws/config`:
   ```bash
   [profile duplo-prod]
   region=us-west-2
   credential_process=duplo-jit aws --admin --host https://prod.duplocloud.net --interactive

   [profile PROFILE_NAME]
   region=DUPLO_REGION
   credential_process=duplo-jit aws --admin --host https://duplo.customer-dns.com --interactive
   ```

2. Export necessary environment variables:
   ```bash
   export USER_EMAIL="user_name@duplocloud.net"
   export DEVOPS_MGR_API_KEY="your_api_key"
   ```

3. Clone the infrastructure repo and run the update:
   ```bash
   git clone https://github.com/duplocloud-internal/duplo-infra.git
   cd duplo-infra/automation
   AWS_PROFILE=PROFILE_NAME VERSION=branch/release_feb_2024/latest UI_VERSION=branch/release_feb_2024/latest ./update-duplo.sh
   ```

This will update Duplo-master to the latest July release.

---

## 8. Update Customer ID

Update the customer ID in the configuration.

### Steps:
1. Navigate to `duplo-infra/single-runs/customer-id`.
2. Run the following command:
   ```bash
   AWS_PROFILE=radiantlogic ./config-customer-id.sh compare xxxxx
   ```
   Replace `xxxxx` with the actual customer ID.
   (You can get customer ID from [DevOps Manager](https://devopsmgr.prod-apps.duplocloud.net/))

---

## 9. Log In to Duplo-Master via RDP

To make updates to Duplo-master, log in via RDP.

### Steps:
1. Start an AWS SSM session and port forward:
   ```bash
   aws ssm start-session --target INSTANCE_ID --document-name AWS-StartPortForwardingSession --parameters portNumber="3389",localPortNumber="56789" --profile PROFILE_NAME
   ```

2. Connect to the instance using RDP with:
   - **Username**: `Administrator`
   - **Password**: `DuploAA007`

---

## 10. Update Duplo-PortalSvcs Instance

Update the Duplo-PortalSvcs instance as required.

### Steps:
1. Use AWS Session Manager to connect.
2. Run the following commands:
   ```bash
   sudo su
   yum update
   sed -i 's#-H fd://#-H fd:// -H tcp://0.0.0.0:4243#' /lib/systemd/system/docker.service
   systemctl daemon-reload
   service docker stop
   service docker start
   reboot -f
   ```

---

## 11. Update Duplo-Master to the Latest July Release

Ensure that Duplo-master is up-to-date by running the latest release update.

### Steps:
1. Navigate to `duplo-infra/automation
2 Run the following command:
  ```bash
  AWS_PROFILE=PROFILE_NAME VERSION=branch/release_jul_2024/latest UI_VERSION=branch/release_jul_2024/latest ./update-duplo.sh
  ```

---

## 12. Disable Secure Time Seeding

Disable secure time seeding to avoid clock synchronization issues.

### Steps:
1. Navigate to `duplo-infra/automation`.
2. Run the following command:
   ```bash
   AWS_PROFILE=profile-name ./disable-secure-time-seeding.sh
   ```

---

## 13. Install OpenVPN Server

Install OpenVPN for secure access.

### Steps:
1. Subscribe to the OpenVPN Access Server from the AWS Marketplace. [Subscription Link](https://aws.amazon.com/marketplace/server/procurement?productId=fe8020db-5343-4c43-9e65-5ed4a825c931).
2. Once the subscription is processed and the server is operational, go to the VPN tab under "System Settings" and provision the VPN stack.

**Fix VpnAuthManager Loop Issue**:
- Update the `sshd_config` file on the OpenVPN server:
   ```bash
   sudo su
   vim /etc/ssh/sshd_config
   ```
   Add the following lines at the end:
   ```bash
   HostKeyAlgorithms +ssh-rsa
   PubkeyAcceptedKeyTypes +ssh-rsa
   ```

- Save and run:
   ```bash
   sshd -t
   service sshd restart
   ```

---

## 14. Add Certificates in Duplo Portal for Logging and Monitoring

Add certificates for logging and monitoring services in the Duplo portal.

### Steps:
1. Navigate to `Administrator -> Plan -> Default -> Certificates`.
2. Click "Add", provide the ARN of the RSA 2048 certificate (created in Step 2), and click "Create".

---

## 15. Enable Logging and Monitoring

Enable logging and monitoring for better observability.

### Steps:
1. Go to `Administrator -> Observability -> Settings`.
2. In the Logging tab, enable logging.
3. In the Monitoring tab, enable monitoring.

---

## Conclusion

Following this guide will help you successfully deploy Duplo Portal on AWS, configure security, set up VPN access, and enable observability. With this setup, you’ll have a secure and scalable environment ready for production or testing.
```
