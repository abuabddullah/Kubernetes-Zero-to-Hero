# Kubernetes-Zero-to-Hero
Creating this repo with an intent to make Kubernetes easy for begineers. This is a work-in-progress repo.

## Kubernetes Installation Using KOPS on EC2

### Create an EC2 instance or use your personal laptop.

Dependencies required 

1. Python3
2. AWS CLI
3. kubectl

###  Install dependencies


Add the new Kubernetes repository:
```
sudo apt-get update
sudo apt-get install -y ca-certificates curl apt-transport-https

```

```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

```

```
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

```
sudo apt-get update
sudo apt-get install -y kubectl

```

Install AWS CLI (Ubuntu 24.04 compatible):

```
sudo snap install aws-cli --classic
```
```
export PATH="$PATH:/home/ubuntu/.local/bin/"
```

### Install KOPS (our hero for today)

```
curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64

chmod +x kops-linux-amd64

sudo mv kops-linux-amd64 /usr/local/bin/kops

kops  # test if install properly
```

### Install kubectl

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

kubectl # test if install properly
```

### Provide the below permissions to your IAM user. If you are using the admin user, the below permissions are available by default

1. AmazonEC2FullAccess
2. AmazonS3FullAccess
3. IAMFullAccess
4. AmazonVPCFullAccess

### Set up AWS CLI configuration on your EC2 Instance or Laptop.

Run `aws configure`

## Kubernetes Cluster Installation 

Please follow the steps carefully and read each command before executing.

### Create S3 bucket for storing the KOPS objects.

```
aws s3api create-bucket   --bucket k8s-my-kops-state-store   --region us-east-1

export KOPS_STATE_STORE=s3://k8s-my-kops-state-store
```
```
ubuntu@ip-172-31-27-116:~$ aws s3api create-bucket   --bucket k8s-my-kops-state-store   --region us-east-1
{
    "Location": "/k8s-my-kops-state-store",
    "BucketArn": "arn:aws:s3:::k8s-my-kops-state-store"
}
```

### Create the cluster 

```
kops create cluster --name=example.k8s.local --state=${KOPS_STATE_STORE} --zones=us-east-1a --node-count=1 --node-size=t2.micro --master-size=t2.micro  --master-volume-size=8 --node-volume-size=8
```
```
ubuntu@ip-172-31-27-116:~$ kops create cluster --name=example.k8s.local --state=${KOPS_STATE_STORE} --zones=us-east-1a -
-node-count=1 --node-size=t2.micro --master-size=t2.micro  --master-volume-size=8 --node-volume-size=8
************************
***********************************
Cluster configuration has been created.

Suggestions:
 * list clusters with: kops get cluster
 * edit this cluster with: kops edit cluster example.k8s.local
 * edit your node instance group: kops edit ig --name=example.k8s.local nodes-us-east-1a
 * edit your control-plane instance group: kops edit ig --name=example.k8s.local control-plane-us-east-1a

Finally configure your cluster with: kops update cluster --name example.k8s.local --yes --admin
```

### Important: Edit the configuration as there are multiple resources created which won't fall into the free tier.

```
kops edit cluster example.k8s.local
```

Step 12: Build the cluster

```
kops update cluster example.k8s.local --yes --state=s3://k8s-my-kops-state-store
```

This will take a few minutes to create............

After a few mins, run the below command to verify the cluster installation.

```
kops validate cluster example.k8s.local
```
Delete the cluster
```
kops delete cluster --name=example.k8s.local --state=s3://k8s-my-kops-state-store --yes
```
চেক করো ডিলিট হয়েছে কি না:
```
kops get cluster --name example.k8s.local --state=s3://k8s-my-kops-state-store
```
যদি "No cluster found" বলে → ডিলিট সাকসেস!
*********************************************************************************
তুমি `kops delete cluster --name=example.k8s.local --state=s3://k8s-my-kops-state-store --yes` রান করেছ – এটা সাকসেসফুল হয়েছে।

### কেন নিশ্চিত হতে পারো যে টাকা আর কাটবে না?

`kops delete cluster --yes` করলে AWS-এ যা যা তৈরি হয়েছিল, সব ডিলিট করে দেয়:
- EC2 instances (মাস্টার + নোড)
- Auto Scaling Groups
- Elastic Load Balancer (ELB) – যেটা API endpoint ছিল
- EBS volumes (৮ GB করে)
- Security Groups, Route Tables, Subnets (যদি dedicated VPC হয়)
- IAM roles/policies (ক্লাস্টার-স্পেসিফিক)

এগুলো ডিলিট হলে **চার্জ বন্ধ হয়ে যায়** (running instance না থাকলে EC2 চার্জ ০ হয়ে যায়)।  
- Free Tier-এর ৭৫০ ঘণ্টা EC2 t2.micro এখনো তোমার অ্যাকাউন্টে রেস্ট থাকবে (যদি আগে বেশি না খরচ করে থাকো) – পরের প্রজেক্টে ব্যবহার করতে পারো।
- যদি কোনো ছোট রিসোর্স (যেমন orphaned EBS snapshot বা ELB) থেকে যায়, সেটা খুব কম চার্জ (~$0.01–$0.10/মাস) – কিন্তু kops সাধারণত সব ক্লিন করে।

**টাকা কাটা বন্ধ হওয়ার নিশ্চয়তা পাওয়ার সেরা উপায়গুলো** (এখনই চেক করো):

1. **AWS Console-এ চেক করো** (সবচেয়ে নির্ভরযোগ্য):
   - EC2 > Instances: "example.k8s.local" ট্যাগযুক্ত কোনো running/stopped instance নেই তো? (Stopped হলেও চার্জ হয় না, শুধু EBS চার্জ হতে পারে – কিন্তু kops delete volumes ডিলিট করে)
   - EC2 > Load Balancers: কোনো ELB নামে "api-example-k8s-local..." আছে কি না? না থাকলে ভালো।
   - VPC > Your VPCs: কোনো extra VPC আছে কি না যেটা ক্লাস্টারের জন্য তৈরি হয়েছিল? (যদি থাকে, ম্যানুয়ালি ডিলিট করো – কিন্তু kops সাধারণত VPC ডিলিট করে না যদি shared হয়)

2. **Billing Dashboard চেক করো**:
   - AWS Console > Billing > Bills (বা Cost Explorer)
   - Last 24 hours / Current month-এ EC2, ELB, EBS-এর চার্জ দেখো।
   - যদি ০ বা খুব কম (~$0.00–$0.50) দেখায় → টাকা কাটা বন্ধ।
   - Cost Explorer > Filter by Service: EC2 → Group by Usage Type → দেখো কোনো "BoxUsage:t2.micro" running আছে কি না।

3. **S3 Bucket-এর অবস্থা**:
   - `kops delete cluster` **S3 bucket ডিলিট করে না** – শুধু bucket-এর ভিতরের cluster state files (যেমন clusters/example.k8s.local/config) ডিলিট করে।
   - Bucketটা এখনো আছে (যদি তুমি ম্যানুয়ালি না ডিলিট করো)।
   - **Bucket-এ চার্জ?** খুব কম (S3 standard storage ~$0.023/GB/মাস) – তোমার ক্লাস্টার state ছোট (কয়েক KB), তাই প্রায় ০। ফ্রি টিয়ারে S3-এরও কিছু ফ্রি অ্যালাউন্স আছে।
   - যদি bucket আর না লাগে: AWS Console > S3 > bucket সিলেক্ট > Empty > Delete। (Empty করার পর delete option আসবে)

### সারাংশ: টাকা কাটবে না (৯৯% নিশ্চিত)

- ক্লাস্টার not found → সব AWS রিসোর্স ডিলিট।
- EC2 running না থাকলে চার্জ ০।
- S3 bucket রাখলে কোনো সমস্যা নেই (খরচ নগণ্য), চাইলে ডিলিট করো।
- পরের ১–২ দিন Billing-এ চেক করে দেখো – যদি কোনো unexpected চার্জ আসে (খুব কম সম্ভাবনা), তাহলে AWS Support-এ ticket খোলো (ফ্রি অ্যাকাউন্টে basic support আছে)।
