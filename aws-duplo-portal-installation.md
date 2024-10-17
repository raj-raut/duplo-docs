Here’s the updated document with the list of AMI IDs, more detailed OpenVPN configuration steps, and a copy button for the S3 URL:

---

# Deploying Duplo Portal on AWS: Updated Guide

This guide outlines the step-by-step process to deploy the Duplo Portal on AWS, including DNS setup, certificate management, CloudFormation deployment, VPN setup, logging, and monitoring. The guide is organized for easy understanding and practical implementation.

---

## 1. Create a Public Hosted Zone in Route 53

Create a public hosted zone for your domain in AWS Route 53. This hosted zone will be used for configuring certificates and CloudFormation stack deployment.

- **Example DNS**: `prod.customer-dns.com`

### Steps:

1. Go to **AWS Route 53**.
2. Create a public hosted zone for your domain (e.g., `prod.customer-dns.com`).

---

## 2. Request Certificates from AWS Certificate Manager (ACM)

Two SSL certificates are required:  
- **Certificate 1**: ECDSA P-384 for the portal domain (e.g., `*.prod.customer-dns.com`).  
- **Certificate 2**: RSA 2048 for logging and monitoring subdomains (wildcard to cover all subdomains).

### Why ECDSA and RSA?
- **ECDSA P-384**: Strong security with smaller key sizes.  
- **RSA 2048**: Ensures robust encryption for logging and monitoring services.

### Steps:

1. Go to **AWS Certificate Manager**.
2. Request an **ECDSA P-384** certificate for your domain.
3. Request an **RSA 2048** wildcard certificate (`*`) to cover all subdomains.

---

## 3. Deploy Duplo Portal Using AWS CloudFormation

We will deploy the Duplo Portal using a CloudFormation stack.

### S3 URL for Template:

[Copy S3 URL](https://duplo-setup-resources.s3-us-west-2.amazonaws.com/2023.07/duplo-root-stack.json)  
(Click here to copy the URL.)

### Steps:

1. **Prepare the Template**:
   - Use the S3 template URL provided above.
2. **Specify Stack Parameters**:
   - **Stack Name**: `duplo-root-stack`
   - **VPC CIDR**: `10.220.0.0/16` (for Class B: 220)
   - **Setup URL**: `https://duplo.prod.customer-dns.com`
   - **Master AMI ID**: Select based on your region from the list below:
   
   ### AMI IDs for Regions:
   - `ap-northeast-1`: `ami-0a5a5654af962ef44`
   - `ap-south-1`: `ami-067e59fd8c1c19857`
   - `ap-southeast-2`: `ami-0ced1fcaffdf9f867`
   - `ca-central-1`: `ami-02638e969501cfa82`
   - `eu-west-2`: `ami-0e58b649241a431a1`
   - `us-east-1`: `ami-0b66c0e497e55a61b`
   - `us-east-2`: `ami-03d5f6731f426902e`
   - `us-west-1`: `ami-0a969ed1c0510538d`
   - `us-west-2`: `ami-0d37a83cc0eb387a9`
   
   ### Additional Parameters:
   - **DUPLODNSPRFX**: e.g., `prod.customer-dns.com`
   - **AWS Route 53 Domain ID**: Enter the Hosted Zone ID.
   - **Bastion AMI ID**: Optional, `ami-0d4efc14256385b61` (if using Bastion).
   - **Default Admin**: `user_name@duplocloud.net`
   - **Enable Bastion**: `NO`
   - **Enable ES Role**: `YES`
   
3. **Create the Stack**:
   - In "Stack Failure Options", choose **Preserve Successfully Provisioned Resources**.
   - Click **Create**.

---

## 4. Configure DNS A Record for Load Balancer

Once the CloudFormation stack is deployed, associate the load balancer DNS with the domain.

### Steps:

1. In **Route 53**, add an **A record** for the load balancer's DNS to your hosted zone.
2. Confirm the load balancer is accessible via the domain name.

---

## 5. Adding Duplo Portal to Google Authentication

To allow Google login on the Duplo Portal:

### Steps:

1. Open a browser and navigate to the DNS (e.g., `https://duplo.prod.customer-dns.com`).
2. Verify that Duplo Portal is running.
3. Add the Duplo Portal DNS to **Google Cloud OAuth** for Google login access.  
   (You can request this via Slack for your DNS.)

---

## 6. Register Customer and Environment in DevOps Manager

### Steps:

1. Navigate to **DuploCloud DevOps Manager**.
2. Register your customer.
3. Create a new environment with necessary details.

---

## 7. Setup AWS CLI Profile

Set up the AWS CLI profile for Duplo.

### Steps:

1. In `~/.aws/config`, add the following:
    ```bash
    [profile duplo-prod]
    region=us-west-2
    credential_process=duplo-jit aws --admin --host https://prod.duplocloud.net --interactive
    
    [profile PROFILE_NAME]
    region=DUPLO_REGION
    credential_process=duplo-jit aws --admin --host https://duplo.customer-dns.com --interactive
    ```

2. Export environment variables:
    ```bash
    export USER_EMAIL="user_name@duplocloud.net"
    export DEVOPS_MGR_API_KEY="your_api_key"
    ```

---

## 8. Update Windows Server

### Steps:

1. Start an **AWS SSM session** and port forward:
    ```bash
    aws ssm start-session --target INSTANCE_ID --document-name AWS-StartPortForwardingSession --parameters "portNumber=3389,localPortNumber=56789" --region REGION_NAME --profile PROFILE_NAME
    ```

2. Connect to the instance using RDP and install updates.
3. Close the RDP session after updates.

---

## 9. Clone GitHub Repository for Duplo Portal Update

Clone the DuploCloud infrastructure repo to apply updates.

### Steps:

1. Run:
    ```bash
    git clone https://github.com/duplocloud-internal/duplo-infra.git
    ```

---

## 10. Update Customer ID

### Steps:

1. Navigate to `duplo-infra/single-runs/customer-id`.
2. Run:
    ```bash
    AWS_PROFILE=PROFILE_NAME ./config-customer-id.sh compare xxxxx
    ```
   Replace `xxxxx` with the actual customer ID (retrieve it from DevOps Manager).

---

## 11. Update Duplo-Master to the Latest February Release

### Steps:

1. Navigate to `duplo-infra/automation`.
2. Run:
    ```bash
    AWS_PROFILE=PROFILE_NAME VERSION=branch/release_feb_2024/latest UI_VERSION=branch/release_feb_2024/latest ./update-duplo.sh
    ```

---

## 12. Update Duplo-Master to the Latest July Release

### Steps:

1. Navigate to `duplo-infra/automation`.
2. Run:
    ```bash
    AWS_PROFILE=PROFILE_NAME VERSION=branch/release_jul_2024/latest UI_VERSION=branch/release_jul_2024/latest ./update-duplo.sh
    ```

---

## 13. Disable Secure Time Seeding

To avoid clock synchronization issues:

### Steps:

1. Navigate to `duplo-infra/automation`.
2. Run:
    ```bash
    AWS_PROFILE=PROFILE_NAME ./disable-secure-time-seeding.sh
    ```

---

## 14. Install OpenVPN Server

### Steps:

1. **Subscribe** to the **OpenVPN Access Server** from the [AWS Marketplace](https://aws.amazon.com/marketplace/).
2. **Provision the VPN Stack**:
   - Go to the VPN tab under "System Settings" and configure VPN settings.
   
3. **Fix the VPN Auth Manager Loop Issue**:
    ```bash
    sudo su
    vim /etc/ssh/sshd_config
    ```

4. Add the following lines:
    ```
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedKeyTypes +ssh-rsa
    ```

5. Save and restart SSH:
    ```bash
    sshd -t
    service sshd restart
    ```

---

## 15. Add Certificates in Duplo Portal for Logging and Monitoring

### Steps:

1. Go to **Administrator** -> **Plan** -> **Default** -> **Certificates**.
2. Add the **ARN** of the RSA 2048 certificate created earlier.

---

## 16. Enable Logging and Monitoring

### Steps:

1. Navigate to **Administrator** -> **Observability** -> **Settings**.
2. Enable both **Logging** and **Monitoring**.

---

## Conclusion

By following this guide, you can successfully deploy the Duplo Portal on AWS, set up security, configure VPN access, and enable observability for your environment.

--- 

Let me know if this version meets your requirements or if further adjustments are needed!
