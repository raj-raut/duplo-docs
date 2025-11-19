# **DuploCloud Production Deployment Runbook**

This runbook provides a comprehensive, enterprise‑grade, end‑to‑end procedure to deploy a DuploCloud environment in a customer AWS account. The workflow encapsulates snapshot migration, AMI creation, DNS delegation, certificate issuance, infrastructure deployment via CloudFormation, load balancer DNS mapping, DevOps Manager onboarding, and platform upgrade.

---

## **1. Share AMI Snapshots from Duplo-Prod AWS Account**

**Source: Duplo-Prod AWS Account**
- Navigate to **EC2 → Snapshots**.
- Identify and select the required snapshots:
  - **DuploMaster AMI snapshot**
  - **Duplo Portal Services AMI snapshot**
- Perform: **Actions → Modify Permissions → Share with Customer AWS Account ID**.

---

## **2. Copy Shared Snapshots and Create AMIs (Customer AWS Account)**

### **Step 2.1: Copy Snapshot with Customer KMS Encryption**
- Switch to the **correct AWS region**.
- Navigate to **EC2 → Snapshots → Private Snapshots**.
- Select the snapshot shared from Duplo-Prod.
- **Actions → Copy Snapshot**.
- Enable **Encryption** and choose the **Customer‑Managed KMS Key**.
- Initiate snapshot copy.

### **Step 2.2: Create AMI from Snapshot**
- After the snapshot copy completes, select the new snapshot.
- **Actions → Create Image**.
- Provide an appropriate **AMI Name**.
- Create the image.
- AMI will appear under **EC2 → AMIs → Private Images**.

> **Repeat Steps 2.1 and 2.2** for:  
> - DuploMaster AMI  
> - Duplo Portal Services AMI

---

## **3. DNS Delegation Setup**

- Navigate to **Route 53**.
- Create a **Public Hosted Zone** for the delegated subdomain (e.g., `cloud.metricstream.com`).
- Capture the **NS Records** from the hosted zone.
- Delegate these records at the **Root Domain Registrar** or **Parent Zone**.

---

## **4. Issue Public Certificate via ACM**

- Go to **ACM → Request a Certificate**.
- Choose **Public Certificate**.
- Enter domain:  
  `*.cloud.metricstream.com`
- Key Algorithm: **ECDSA P‑384**.
- Submit the request.
- Complete **DNS validation**.

---

## **5. Deploy Duplo Root CloudFormation Stack**

### **Step 5.1: Select Deployment Template**
- Navigate to **CloudFormation → Create Stack**.
- Select **Amazon S3 URL** and enter:
  `https://duplo-setup-resources.s3.us-west-2.amazonaws.com/2025.08/duplo-root-stack.json`

### **Step 5.2: Populate Stack Parameters**
Provide values based on customer architecture and prerequisites:

- **Stack Name:** `Duplo-root-stack`
- **VPC and Subnet CIDRs:**
  - `vpccidr`
  - `SubnetAPublicCidr`
  - `SubnetAPrivateCidr`
  - `SubnetBPublicCidr`
  - `SubnetBPrivateCidr`
- **SetupUrl:** Duplo portal URL (e.g., `https://duplo.cloud.metricstream.com`)
- **MasterAmiId:** AMI from Step 2
- **DUPLODNSPRFX:** Delegated domain prefix (e.g., `cloud.metricstream.com`)
- **AWSEXTRAREGIONS:** Colon‑separated regions manageable by Duplo
- **AWSROUTE53DOMAINID:** Hosted zone ID created in Step 3
- **BastionAmiId:** Leave default
- **DefaultAdmin:** Microsoft login (primary Duplo admin)
- **DefaultEbsEncryption:** `YES`
- **ENABLEBASTION:** `NO`
- **ENABLEESROLE:** `YES`
- **ENABLEJITAWSADMINAPI:** `true`
- **ENABLEPOSTINSTALLLAMBDA:** `NO`
- **GENBOOTSTRAPTOKEN:** `false`
- **Lambda Parameters:**
  - Handler: `index.handler`
  - Memory: `384`
  - Runtime: `python3.12`
  - Bucket: `duplo-setup-resources`
  - Key: `examples/postinstall-terraform.zip`
  - Timeout: `900`
- **PortalCertificateArn:** ACM certificate ARN from Step 4
- **PortalSvcsAmiId:** Portal services AMI from Step 2

### **Step 5.3: Deployment Options**
- In **Stack Failure Options**, enable:  
  **Preserve successfully provisioned resources**.
- Acknowledge all required capabilities.
- Review and submit.

CloudFormation will orchestrate the full Duplo infrastructure deployment.

---

## **6. Configure Route53 A Record for Portal Load Balancer**

- Navigate to **Route53 → Hosted Zone**.
- Create a new record:
  - **Record Name:** Prefix from Duplo portal URL (e.g., `duplo`)
  - **Record Type:** `A`
  - Enable **Alias**
  - **Endpoint Type:** Application or Classic Load Balancer
  - **Region:** Deployed region
  - **Load Balancer:** Select `dualstack.PortalDuplo‑*`
  - Routing Policy: **Simple**

This links the portal DNS to its provisioned load balancer.

---

## **7. Onboard Environment to Duplo DevOps Manager**

Submit the following details to DuploCloud support to onboard the environment:
- DuploMaster EC2 Instance ID
- Duplo Portal Services Instance ID
- Region
- Duplo Portal URL
- AWS Account ID

---

## **8. Upgrade Duplo Portal to Latest Release**

### **Step 8.1: Prepare AWS CLI Profiles**
Update `~/.aws/config`:
```
[profile profile_name]
region=ap-south-1
credential_process=duplo-jit aws --admin --host https://duplo_portal_url --interactive

[profile duplo-prod]
region=us-west-2
```

Update `~/.aws/credentials`:
```
[duplo-prod]
aws_access_key_id = access-key
aws_secret_access_key = secret-key
```

### **Step 8.2: Run Duplo Update Process**
- Unzip **Duplo Infra Master.zip**.
- Navigate to:  
  `duplo-infra-master/single-runs/customer-id`
- Execute configuration comparison:
```
AWS_PROFILE=profile_name ./config-customer-id.sh compare 00305
```

### **Step 8.3: Execute Component Updates**
Navigate to `duplo-infra-master/automation` and run:
```
export USER_EMAIL="devops-manager-user"
export DEVOPS_MGR_API_KEY="devops-manager-api-key"

AWS_PROFILE=profile_name \
UPDATE_UI=true \
UPDATE_AI_UI=true \
UPDATE_BACKEND=true \
UPDATE_AI_STUDIO=true \
AI_STUDIO_VERSION=tag/v0.1.4 \
AI_UI_VERSION=branch/release_jan_2025/latest \
UI_VERSION=branch/release_jan_2025/latest \
VERSION=branch/release_jan_2025/latest \
./update-duplo.sh
```

Disable secure time seeding:
```
AWS_PROFILE=profile_name ./disable-secure-time-seeding.sh
```

Portal is now upgraded and ready for production operations.

---

# **End of Document**

