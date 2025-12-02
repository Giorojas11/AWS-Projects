## **What I Applied**

### **Security Principles**
#### Least Privilege
- Only the minimum permissions required were granted to IAM users, roles, and services.
- Root account usage was avoided for daily tasks.
- Policies were scoped to specific resources rather than broad access.

#### Secure Identity & Access Management
- MFA enabled for both root and administrative accounts.
- Access keys created only when needed and deactivated after use.
- CLI access secured via IAM credentials instead of root credentials.
#### Defense In Depth
- MFA + IAM for identity protection
- S3 public access blocked
- Object Lock for data retention & protection

#### Compliance & Data Governance
- Object Lock enforce regulatory compliance (HIPAA, FINRA, SEC, SOX).
- Infrastructure-as-Code ensures consistent, auditable configurations.

### **Infrastructure-as-Code Benefits**
- Repeatable, predictable deployments - I can deploy this bucket when needed
- Modular Terraform code blocks for maintainability
- Plugin Version Locking through .terraform.lock.hcl
----
## **Write-Up**

I installed Terraform for Windows from https://developer.hashicorp.com/terraform/install. I then created a folder for Terraform (C:\Terraform) and moved Terraform.exe from my Downloads folder to C:\Terraform. I verified the installation using `terraform -help` on my Command Prompt.

<img width="801" height="884" alt="TFinstall" src="https://github.com/user-attachments/assets/579cb8fe-da94-4ea5-a158-eccf5c1cb4c8" />

I then created `main.tf` using `type nul > main.tf`. Terraform uses configuration files to understand our desired infrastructure and impliment it. `main.tf` is the main config file for Terraform. 

<img width="1015" height="519" alt="main tf" src="https://github.com/user-attachments/assets/9fbeaf7b-1a9b-400f-8b9e-7af517e07d8e" />

The code is separated into three blocks to provide modularity, making it easy to update individual blocks as needed. I plan to build upon this in the future. I also adjusted the retention of the bucket and its resources for data governance and compliance. In this case I have it set to 2555 days or 7 years. This is a homelab, I don't have any compliance standards I need to meet but it was good practice to learn about this resource.

Code block used:
`resource "aws_s3_bucket_object_lock_configuration" "my_bucket_object_lock" {
   bucket = aws_s3_bucket.my_bucket.id
    rule {
    default_retention {
      mode = "COMPLIANCE"
      days = 2555
    }
  }
}`

I then ran `terraform init` to boot up Terraform. In this process I learned that all resource blocks must have 2 labels: type and name. I was missing a name label for `aws_s3_bucket_public_access_block`. After creating a name label `terraform init` worked and downloaded the AWS plugin and created a lock file that saves the plugin and its version.

<img width="1075" height="572" alt="init" src="https://github.com/user-attachments/assets/65af03d0-e516-49d3-b153-4e83bd59aa68" />

I ran the `terraform plan` command which allows terraform to compare the infrastructure and resources from main.tf to the existingi infrastructure and resources in my AWS environment. I encountered a `No valid credential sources found` because I did not have credentials from AWS to authenticate and allow Terraform to review my AWS.

<img width="1084" height="367" alt="initerror" src="https://github.com/user-attachments/assets/74674343-f2a2-4c95-a88e-4aa5e72964a6" />

So, I installed AWS CLI locally using `C:\Terraform>msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi`. I verified the installation using `aws --version` which gave me `aws-cli/2.32.7 Python/3.13.9 Windows/11 exe/AMD64`. Following the least-privilege and separation of duties principles, I did not want to use my ROOT user for this project and future operational tasks, I created a user with administrator privileges via the console. 

I created GROJAS-IAM-ADMIN with console access, a secure password, and added the AdministratorAccess permissions policy. I also enabled MFA on this account and my Root user. This provides stronger authentication requiring something I know (password) and something I have (authenication app). This is defense-in-depth, an extra layer of protection has been added at the IAM level of my AWS environment.

<img width="1679" height="531" alt="grojasadminuser" src="https://github.com/user-attachments/assets/0006df29-3c53-4604-9ebf-5412efa43cea" />

With my Admin account set up, I created an Access Key which will allow me to authenticate and use the AWS CLI locally. I went to AWS > IAM > Users > GROJAS-IAM-ADMIN > Security Credentials > Create Access Key to create the Access Key.

Now that I can use AWS CLI, running `terraform plan` successfully reviews my AWS environment and what changes will be made. To run this plan, I used `terraform apply` which actually runs the changes.

<img width="1543" height="949" alt="TFplan" src="https://github.com/user-attachments/assets/2bbfd666-769b-4d0a-8fde-f767714d812e" />

<img width="786" height="320" alt="plancreated" src="https://github.com/user-attachments/assets/50dded48-cd1e-4c3d-baa2-41fde410b33b" />

**This created my S3 bucket and its permission settings.**

<img width="702" height="194" alt="bucketconsole" src="https://github.com/user-attachments/assets/8e709dbf-2b23-4fa5-a7ae-354e01f09625" />

<img width="1576" height="207" alt="publicaccessconsole" src="https://github.com/user-attachments/assets/fa38245b-58d0-431f-b893-e81daad8d51e" />

Now that the S3 bucket is created, I wanted to attempt uploading a file using `aws_s3_object` in my main.tf file. Since I have image.png saved to C:\Terraform, I don't have to specify the local file path but if it were in Downloads, for example, the source would be C:\Users\Downloads\image.png. I used `terraform plan` and `terraform apply` with the updated `main.tf`. I verified the changes were made via CLI and console.

<img width="1537" height="33" alt="imageconsole" src="https://github.com/user-attachments/assets/e87c6c47-dd4e-4481-bb8b-acd1ef3e9d01" />

The image used:

<img width="496" height="640" alt="gnomey" src="https://github.com/user-attachments/assets/95577d21-85a7-4430-9b75-1f281ae13ebf" />

To finish this project, I decided to delete the S3 bucket using `terraform destroy`. A big advantage to using Terraform, and IaC in general, is the ability to stand up/tear down cloud resources quickly and consistently. To prevent an additional attack vector while not using the AWS CLI, I will also be deactivating the Access Key. I can recreate one easily next time I need to access the CLI.

----

## Conclusion

This project reinforced the importance of integrating security directly into cloud infrastructure using Infrastructure-as-Code. By deploying and configuring an Amazon S3 bucket with Terraform, I applied key cloud security principles such as least privilege, defense-in-depth, MFA, IAM, and compliance-driven data governance.

Through hands-on implementation, I gained practical experience in:
- Managing IAM identities and enabling MFA to protect accounts
- Configuring S3 with public access blocks, Object Lock, and retention policies
- Using Terraform to automate, standardize, and audit deployments
- Troubleshooting issues like credential errors and misconfigured resources

Overall, this project demonstrates how careful planning and secure configurations reduce attack surfaces, improve compliance, and ensure predictable, auditable deployments. It highlights the value of combining cloud engineering and security skills—essential for modern cybersecurity and cloud operations roles.


