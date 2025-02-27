# 50 Terraform Interview Questions & Answers (Intermediate to Expert Level)

## Infrastructure as Code Fundamentals

### 1. What is the difference between Terraform state locking and state versioning?

Why Only DynamoDB for Terraform State Locking?

3️⃣ Fully Managed & Highly Available – No need to manage servers, patch software, or handle failover—DynamoDB is AWS-managed and replicated across AZs.

4️⃣ Automatic Lock Expiry (TTL Feature) – Prevents stale locks if a Terraform process crashes because it has TTL, unlike RDS or file-based locks that require manual cleanup.

5️⃣ Cost-Effective & IAM-Based Security – Cheaper than RDS, scales automatically, and uses IAM policies for fine-grained access control, making it more secure and efficient

**Answer:**
State locking prevents concurrent operations that might corrupt the state file. When a Terraform operation begins that could write to the state, it locks the state file to prevent other processes from acquiring the lock and potentially corrupting the state.

State versioning maintains a history of state file changes. This allows for recovering previous states if a mistake is made.

**Example:**
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "path/to/my/key"
    region         = "us-west-2"
    dynamodb_table = "terraform-locks" # For state locking
  }
}
```

In this example, the DynamoDB table provides locking functionality, while S3 versioning (enabled on the bucket) provides state versioning.

### 2. How does Terraform handle dependencies between resources, and what happens if you remove a dependency?

**Answer:**
Terraform automatically creates a dependency graph of all resources in your configuration. This helps determine:
✔ Order of resource creation (which resources must be created first).
✔ Parallelization (which resources can be created at the same time).
✔ Order of resource destruction (which dependencies must be deleted first).

1. **Implicit** - When one resource references attributes of another using interpolation.
resource "aws_vpc" "my_vpc" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "my_subnet" {
  vpc_id = aws_vpc.my_vpc.id  # 🔥 Implicit dependency
  cidr_block = "10.0.1.0/24"
}

2. **Explicit** - When using the `depends_on` argument.
resource "aws_instance" "my_vm" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  depends_on    = [aws_security_group.my_sg]  # 🔥 Explicit dependency
}

resource "aws_security_group" "my_sg" {
  name = "my-security-group"
}

### 3. Explain the concept of idempotence in Terraform and why it's important.

**Answer:**
Terraform compares the current infrastructure state (terraform.tfstate) with the desired configuration (.tf files) and only applies changes if necessary.

1. It allows for safe, repeatable deployments
2. It enables drift detection and remediation
3. It ensures consistency across environments
4. It makes configurations testable and predictable

Terraform achieves idempotence by:
- Tracking the current state and comparing it to the desired state
- Only making changes necessary to reach the desired state
- Handling dependencies correctly to ensure proper order of operations

**Example:**
Running `terraform apply` twice on the same configuration should only make changes the first time (assuming no external changes).

### 4. What is the difference between `terraform apply` and `terraform apply -auto-approve`?

**Answer:**
- `terraform apply`: Shows the execution plan and requires manual confirmation before making any changes.
- `terraform apply -auto-approve`: Skips the confirmation step and immediately applies the changes.

The difference is important from a safety perspective:
- Without `-auto-approve`, you have a chance to review changes before execution
- With `-auto-approve`, changes apply immediately (useful for CI/CD pipelines but riskier)

**Recommendation:** Never use `-auto-approve` for production environments without thorough testing in a pre-production environment.

### 5. How would you handle sensitive data in Terraform?

**Answer:**
Terraform offers several methods to handle sensitive data:

variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true  # 🔥 Prevents exposure in logs
}

resource "aws_db_instance" "my_db" {
  engine         = "mysql"
  instance_class = "db.t2.micro"
  password       = var.db_password  # Terraform will not show this in CLI output
}

AWS Secrets Manager integration --
- save the password - 
  aws secretsmanager create-secret --name "my-db-password" --secret-string "SuperSecret123!"

- retreive - 
  data "aws_secretsmanager_secret" "db_secret" {    ///This finds the secret based on the name "my-db-password".
  name = "my-db-password" 
  sensitive = true    ////Prevents Terraform from displaying it
  }
  ✔ This only finds metadata about the secret (e.g., ID, ARN).
  ✔ It does NOT return the actual secret value!

  data "aws_secretsmanager_secret_version" "db_secret_version" {  
  secret_id = data.aws_secretsmanager_secret.db_secret.id
  }
  use that - data.aws_secretsmanager_secret_version.db_secret_version.secret_string
   
**Best Practices:**
- Never commit secrets to version control
- Leverage centralized secret management
- Mark variables as sensitive to prevent values appearing in logs
- Encrypt state files at rest and in transit
- Use IAM roles for authentication where possible instead of static credentials

## Terraform State Management

### 6. When would you use state locking, and what happens if you don't?

**Answer:**
You should use state locking whenever multiple users or processes might attempt to modify the same infrastructure concurrently. This is particularly important in team environments or CI/CD pipelines.

Without state locking:
- Multiple simultaneous operations could corrupt the state file
- Race conditions could lead to unexpected outcomes
- Resources might be created when they should be updated, or vice versa
- You risk inconsistency between your state and actual infrastructure

**Example Configuration (AWS S3 + DynamoDB):**
```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state-bucket"
    key            = "path/to/my/state"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

### 7. Explain the purpose of a remote backend and compare at least three options.

**Answer:**
Remote backends store Terraform state files remotely rather than on local disk. They provide:
- Collaboration support for teams
- State locking to prevent corruption
- Secure storage for potentially sensitive data
- A central source of truth for infrastructure state

**Comparison of three common backends:**

1. **S3 Backend (AWS)**:
   - Pros: Inexpensive, versioning, encryption, highly durable
   - Cons: AWS-specific, requires separate DynamoDB table for locking
   - Best for: AWS-focused teams

2. **Terraform Cloud/Enterprise**:
   - Pros: Managed service, CI/CD integration, granular permissions, centralized management
   - Cons: Cost (Enterprise), additional system to manage
   - Best for: Teams needing governance features or a SaaS solution

3. **Azure Storage**:
   - Pros: Blob versioning, encryption, Azure-native integration
   - Cons: Azure-specific, more complex setup than S3
   - Best for: Azure-focused teams

**Example (Terraform Cloud):**
```hcl
terraform {
  backend "remote" {
    organization = "my-company"
    workspaces {
      name = "my-app-prod"
    }
  }
}
```

### 8. What steps would you take to migrate from a local backend to a remote backend?

**Answer:**
1. **Create the remote backend infrastructure** (e.g., S3 bucket, Azure storage account)
2. **Configure the backend in your Terraform code**:
   ```hcl
   terraform {
     backend "s3" {
       bucket = "my-terraform-state"
       key    = "path/to/my/key"
       region = "us-west-2"
     }
   }
   ```
3. **Initialize with migration flag**: Run `terraform init -migrate-state` which will:
   - Prompt for confirmation
   - Copy the local state to the remote backend
   - Remove the local state file
4. **Verify migration success**: 
   - Check the remote backend contains the state
   - Run `terraform plan` to confirm no changes are detected
5. **Update documentation and team processes**
6. **Commit backend configuration to version control**

**Best Practice:** Test this process in a non-production environment first.

### 9. How would you handle a corrupted Terraform state file?

**Answer:**
1. **Stop all operations**: Ensure no other Terraform operations are running
2. **Check for backups**:
   - If using a remote backend with versioning (like S3), restore a previous version
   - Check for local backups (e.g., `terraform.tfstate.backup`)
3. **Use state recovery commands**:
   - `terraform state pull > terraform.tfstate` to retrieve the current state
   - Edit the state file if needed (as a last resort)
   - `terraform state push terraform.tfstate` to upload the fixed state
4. **Perform a refresh**: `terraform refresh` to reconcile the state with reality
5. **Run a plan**: Verify the state appears correct with `terraform plan`
6. **Implement preventive measures**:
   - Use remote backends with versioning
   - Establish proper locking mechanisms
   - Set up team processes for state management

**Emergency Recovery (last resort)**:
If no backups exist, you might need to:
```
# Create an empty state file
echo "{\"version\": 4, \"terraform_version\": \"1.3.0\", \"serial\": 0, \"lineage\": \"\", \"outputs\": {}, \"resources\": [] }" > terraform.tfstate

# Import existing resources
terraform import aws_instance.example i-1234567890abcdef0
```

### 10. Explain how to use workspaces in Terraform and when they should be used.

**Answer:**
Terraform workspaces allow you to manage multiple distinct states using the same configuration files. They're managed with the `terraform workspace` commands.

**Key Commands:**
```bash
terraform workspace new dev      # Create new workspace
terraform workspace select prod  # Switch to workspace
terraform workspace list         # List workspaces
terraform workspace show         # Show current workspace
```

**Example Usage in Configuration:**
```hcl
resource "aws_instance" "example" {
  instance_type = terraform.workspace == "prod" ? "m5.large" : "t3.micro"
  count         = terraform.workspace == "prod" ? 10 : 1
  
  tags = {
    Environment = terraform.workspace
  }
}
```

**When to Use Workspaces:**
- ✅ For minor environment variations (dev/test/stage)
- ✅ For ephemeral developer environments
- ✅ For feature branch testing environments

**When NOT to Use Workspaces:**
- ❌ For significantly different configurations
- ❌ For production vs. non-production (better to use separate state files)
- ❌ When different teams manage different environments

**Best Practice:** For production environments, use separate configurations with different backend state files rather than workspaces.

## Modules and Structure

### 11. What are the best practices for organizing large Terraform projects?

**Answer:**
For large-scale Terraform projects:

1. **Repository Structure**:
   - Use a monorepo or multiple repos based on team structure
   - Consider using Terragrunt for repo organization

2. **Module Organization**:
   - Create reusable modules for common patterns
   - Separate modules by resource type or functional area
   - Maintain separate modules for each major cloud provider

3. **Environment Segregation**:
   - Use separate state files for each environment
   - Structure as `environments/[env]/[component]`

4. **Component Separation**:
   - Split large infrastructures into logical components
   - Example: networking, data storage, compute, security, etc.

**Example Directory Structure:**
```
terraform-infrastructure/
├── modules/                    # Reusable modules
│   ├── networking/
│   ├── database/
│   └── kubernetes/
├── environments/               # Environment-specific configurations
│   ├── dev/
│   │   ├── vpc/
│   │   ├── databases/
│   │   └── services/
│   ├── staging/
│   │   ├── vpc/
│   │   ├── databases/
│   │   └── services/
│   └── prod/
│       ├── vpc/
│       ├── databases/
│       └── services/
└── global/                     # Global/shared resources
    ├── iam/
    ├── dns/
    └── monitoring/
```

### 12. How do you effectively use the Terraform Registry and what differentiates a good module?

**Answer:**
The Terraform Registry is a repository of modules shared by the community and HashiCorp, making it easier to reuse well-architected infrastructure code.

**Effective Usage:**
1. Search for modules that match your requirements
2. Evaluate module quality (stars, usage, age, docs)
3. Pin to specific versions using semantic versioning
4. Contribute improvements back to the community

**Example Usage:**
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "3.14.0"

  name            = "my-vpc"
  cidr            = "10.0.0.0/16"
  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
}
```

**Characteristics of Good Modules:**
1. **Clearly defined purpose**
2. **Well-documented** - README with examples, inputs, outputs
3. **Proper versioning** - semantic version tags
4. **Sensible defaults** but highly configurable
5. **Validation** of inputs and proper error handling
6. **Complete examples** showing various use cases
7. **Automated tests**
8. **Security considerations** built-in

### 13. How would you deal with a module that needs to be different between environments, beyond simple variable changes?

**Answer:**
I would use Terragrunt and modules to handle environment-specific differences. Instead of just using variables, I would create separate module configurations for each environment. With Terragrunt, I can dynamically load different modules or configurations based on the environment. This avoids code duplication while keeping infrastructure consistent.

terragrunt-infra/
│── modules/                    # Reusable Terraform modules
│   ├── vpc/                    # VPC module
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   ├── ec2/                    # EC2 module
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│
│── terragrunt/                 # Terragrunt configurations
│   ├── test/                   # Test environment
│   │   ├── terragrunt.hcl
│   ├── stage/                  # Stage environment
│   │   ├── terragrunt.hcl
│   ├── prod/                   # Prod environment
│   │   ├── terragrunt.hcl
│
│── terragrunt.hcl              # Global config for all environments

# root terragrunt.hcl -     //thats how i can call modules in terragrant 
  terraform {
  source = "${get_repo_root()}/modules//"
}

remote_state {
  backend = "s3"
  config = {
    bucket         = "my-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-lock"
  }
}

2️⃣ terragrunt/test/terragrunt.hcl (Test Environment)
include {
  path = find_in_parent_folders()
}

terraform {
  source = "../../modules/"
}

inputs = {
  vpc_cidr_block = "10.0.1.0/24"
  instance_type  = "t2.micro"
}

3️⃣ terragrunt/stage/terragrunt.hcl (Stage Environment)
include {
  path = find_in_parent_folders()
}

terraform {
  source = "../../modules/"
}

inputs = {
  vpc_cidr_block = "10.0.2.0/24"
  instance_type  = "t3.medium"
}

4️⃣ terragrunt/prod/terragrunt.hcl (Prod Environment)
include {
  path = find_in_parent_folders()
}

terraform {
  source = "../../modules/"
}

inputs = {
  vpc_cidr_block = "10.0.0.0/16"
  instance_type  = "t3.large"
}

5️⃣ modules/vpc/main.tf (Reusable VPC Module)
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr_block
  tags = {
    Name = "VPC-${terraform.workspace}"
  }
}
✅ modules/vpc/outputs.tf
variable "vpc_cidr_block" {
  description = "CIDR block for the VPC"
  type        = string
}
output "vpc_id" {
  value = aws_vpc.main.id
}
✅ modules/vpc/variables.tf
variable "vpc_cidr_block" {
  description = "CIDR block for the VPC"
  type        = string
}

🚀 Deploying Each Environment
Run these commands inside each environment:
cd terragrunt/test
terragrunt apply --auto-approve

cd ../stage
terragrunt apply --auto-approve

cd ../prod
terragrunt apply --auto-approve

📌 Why Is This Setup Good?
✅ No duplicate Terraform code (everything is modular).
✅ Different environments, different configurations (VPC & EC2 settings).
✅ State files are isolated (each env has its own S3 state file).
✅ Easy deployment with terragrunt apply.

🚀 Now your infrastructure is clean, scalable, and easy to manage! 🎉

### 15. How do you handle versioning of your Terraform modules?

**Answer:**
Module versioning is crucial for stability and controlled upgrades. Here's a comprehensive approach:

🌟 Scenario - You have:
Module Repo (terraform-modules.git) → Contains your EC2 module.
Project Repo → Uses the module to create an EC2 instance.

Now, you want to:
Version your module correctly using Git.
Check version history and see what changed.
Use a specific version in your Terraform project.

1️⃣ Versioning Your Module
Step 1: Tagging the First Version (v1.0.0)
You finished your first working version of the EC2 module, so you tag it:
cd /path/to/terraform-modules
git tag v1.0.0
git push origin v1.0.0  # Push the tag to GitHub

✅ Now, v1.0.0 is saved in GitHub as a versioned release.
Step 2: Making Changes (Adding SSH Key)
Now, you add SSH key support to the module.
Before pushing the changes, you create a new tag:
git tag v1.0.1
git push origin v1.0.1  # Push the new tag to GitHub

✅ Now, you have two versions (v1.0.0 and v1.0.1) available in GitHub.
2️⃣ Listing Versions & Checking Changes
Step 3: List All Versions
To see all tagged versions:
git tag   //Lists all tags that exist locally.   ///If you tagged a version but haven't pushed it yet, it will only appear here.
This will show:
v1.0.0
v1.0.1

If you want to see tags in GitHub, run:
git ls-remote --tags origin  ///Shows all tags that exist in the remote repo.  ///If a tag appears locally but not in this list, you likely forgot to push it.

Step 4: Check What Changed in Each Version
To see what changed between v1.0.0 and v1.0.1:
git diff v1.0.0 v1.0.1
It will show something like:
+ variable "ssh_key" {
+   type = string
+ }
✅ This tells you that v1.0.1 introduced the SSH key feature.

If you just want to see commit messages per version, run:
git log v1.0.0..v1.0.1 --oneline
This shows all commits between v1.0.0 and v1.0.1.

3️⃣ Using a Specific Module Version in Terraform
Now, in your Terraform project repo, you can use a specific module version.
Use the first version (v1.0.0):
module "ec2" {
  source = "git::https://github.com/my-org/terraform-modules.git//ec2?ref=v1.0.0"
}

Switch to the new version (v1.0.1):

module "ec2" {
  source = "git::https://github.com/my-org/terraform-modules.git//ec2?ref=v1.0.1"
}

Update Terraform to fetch the new module version:
terraform get -update
terraform init -upgrade

✅ Now, your Terraform project is using v1.0.1, which includes SSH key support.

## Advanced Configuration Techniques

### 16. Explain the concept of count vs for_each, and when you would choose one over the other.

**Answer:**
Both `count` and `for_each` allow you to create multiple instances of a resource, but they have different strengths:

**Count**:
```hcl
resource "aws_instance" "server" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  tags = {
    Name = "server-${count.index}"
  }
}
```
✅ Good for:
✔ Same instance type, AMI, and configuration.
✔ Simple, predictable structure.
✔ Order matters (like server-0, server-1, server-2).

❌ Problem with Deletion:
If you remove server-0 by reducing count = 2, Terraform will recreate server-1 as server-0 and server-2 as server-1.

**For_each with a map**:
```hcl
resource "aws_instance" "server" {
  for_each      = {
    web  = "t2.micro"
    api  = "t2.small"
    db   = "t2.medium"
  }
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = each.value
  
  tags = {
    Name = "server-${each.key}"
  }
}
```
✅ Good for:
✔ Unique instance types (or other attributes).
✔ Resources won’t be recreated if one is removed.
✔ Useful for maps or sets of dynamic values.

✅ How Deletion Works (for_each)

If you remove "web" from the map, only server-web is deleted.
"server-api" and "server-db" remain untouched.
❌ Why Not Use for_each All the Time?

More complex than count – requires a map/set instead of just a number.
Extra management overhead – better for unique configurations, not simple duplication.
Terraform doesn’t support for_each on lists, so you need to convert lists to sets/maps (toset() or {}), adding complexity.

📌 Example: Using for_each with toset
resource "aws_instance" "server" {
  for_each = toset(["app1", "app2", "app3"])  # Convert list to a set
  ami      = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "server-${each.key}"  # Creates "server-app1", "server-app2", "server-app3"
  }
}


**When to use Count**:
- When instances are identical except for a simple incrementing number
- When working with simple lists where order is important
- For simple scaling of homogeneous resources

**When to use For_each**:
- When resources have unique names/identifiers
- When each instance has different configurations
- When removing a middle item shouldn't affect other resources
- When order doesn't matter, but stability does

**Key advantage of for_each:** When you remove an item from the middle of a count list, all higher-indexed resources are recreated. With for_each, only the removed item is affected.

### 17. What are Terraform's meta-arguments and how would you use them?
Meta-arguments in Terraform don’t define resources but instead change their behavior—like how many to create, conditions for creation, or dependencies.

Terraform Meta-Arguments Guide

1️⃣ count: Create Multiple Identical Instances
Creates a specified number of identical resources.✅ Best for: When you need identical resources but with a unique index.
resource "aws_instance" "server" {
  count         = 3  # Creates 3 instances
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "server-${count.index}"  # "server-0", "server-1", "server-2"
  }
}
📌 Issue with Deletion: If you remove count = 2, Terraform may recreate instances with shifted indexes.

2️⃣ for_each: Create Multiple Unique Resources
Creates multiple resources dynamically using a map or set.✅ Best for: When each resource has unique attributes.
🟢 Using a Map (Key-Value Pairs)
resource "aws_route53_record" "www" {
  for_each = {
    us-east-1 = "10.0.1.0"
    us-west-2 = "10.0.2.0"
  }
  zone_id = aws_route53_zone.primary.zone_id
  name    = "www.example.com"
  type    = "A"
  ttl     = 300
  records = [each.value]
}
✅ Creates:
www-us-east-1 → IP 10.0.1.0
www-us-west-2 → IP 10.0.2.0

🟢 Using a Set (List of Unique Names)
resource "aws_instance" "server" {
  for_each = toset(["app1", "app2", "app3"])  # Converts list to a set
  ami      = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "server-${each.key}"  # "server-app1", "server-app2", "server-app3"
  }
}
📌 Advantage: If you remove "app1", only that instance is deleted, without affecting others.

3️⃣ depends_on: Define Dependencies
Ensures a resource is created only after another resource exists.✅ Best for: When Terraform doesn’t detect dependencies automatically.
resource "aws_s3_bucket" "example" {
  bucket = "my-unique-bucket-name"
}
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  depends_on    = [aws_s3_bucket.example]  # Ensures S3 bucket is created first
}
📌 Without depends_on, Terraform might try to create the EC2 before the S3 bucket exists.

4️⃣ lifecycle: Manage Resource Behavior
Controls how Terraform creates, updates, and destroys resources.✅ Best for: Preventing unwanted changes, ensuring smooth updates.
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  lifecycle {
    create_before_destroy = true  # Avoids downtime by creating a new instance first
    prevent_destroy       = true  # Prevents accidental deletion
    ignore_changes        = [tags]  # Ignore tag changes (e.g., manually updated tags)
  }
}
📌 Behavior:
If Terraform tries to delete this instance, it fails due to prevent_destroy.
Tag changes won’t trigger updates.

5️⃣ provider: Use a Specific Provider Configuration
Overrides the default provider for a specific resource.✅ Best for: Multi-region or multi-cloud deployments.
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}
resource "aws_instance" "example" {
  provider      = aws.west  # Use the "west" provider instead of default
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
📌 Behavior:
This instance only runs in us-west-2, even if the default provider is us-east-1.

### 18. Explain how the `terraform_remote_state` data source works and when it should be used.
**Answer:**
The `terraform_remote_state` data source retrieves the state from another Terraform configuration, allowing you to use outputs from one Terraform project as inputs to another.

Example Scenario
Networking Team creates VPC, Subnets, Security Groups, and stores Terraform state remotely (e.g., in an S3 bucket or Terraform Cloud).
App Team needs the App Subnet ID to deploy EC2 instances but shouldn’t manage the networking itself.
Solution: The Networking Team outputs the app_subnet_id, and the App Team fetches it using terraform_remote_state.

Networking Team - Creating VPC with Two Subnets
terraform {
  backend "s3" {  
    bucket = "my-terraform-state"
    key    = "networking/terraform.tfstate"  
    region = "us-east-1"
  }
}
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
resource "aws_subnet" "app_subnet" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}
resource "aws_subnet" "db_subnet" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.2.0/24"
}

# Outputs for other teams
output "vpc_id" {
  value = aws_vpc.main.id
}

output "app_subnet_id" {
  value = aws_subnet.app_subnet.id
}

output "db_subnet_id" {
  value = aws_subnet.db_subnet.id
}
📌 Networking state is stored in: s3://my-terraform-state/networking/terraform.tfstate

- App Team - Fetching the App Subnet Using terraform_remote_state
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  subnet_id     = data.terraform_remote_state.networking.outputs.app_subnet_id  # Fetching App Subnet

  tags = {
    Name = "App-Instance"
  }
}
📌 The App Team now dynamically fetches the app_subnet_id without needing to modify networking infrastructure.
**When to use:**
1. To share information between separate Terraform configurations
2. To maintain separation of concerns (e.g., networking vs. applications)
3. When different teams are responsible for different parts of infrastructure
4. When you need to break a large configuration into smaller, more manageable pieces

### 20. Describe the purpose of `terraform.tfvars`, `terraform.tfvars.json`, and `*.auto.tfvars` files.
**Answer:**
These files provide values for your defined variables in a Terraform configuration:

Scenario - We want to deploy an EC2 instance in a VPC with a dynamic subnet and instance type, where the configuration differs for prod and test environments.

- main.tf (Defines EC2 Instance, VPC, and Subnet)
provider "aws" {
  region = "us-east-1"
}
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.instance_type
  subnet_id     = var.subnet_id

  tags = {
    Name = "${var.environment}-app"
  }
}

- variables.tf (Defines Variables Without Values)
variable "instance_type" {
  type = string
}
variable "vpc_id" {
  type = string
}
variable "subnet_id" {
  type = string
}
variable "environment" {
  type = string
}

- Environment-Specific Variable Files
- prod.tfvars (For Production Environment)
instance_type = "t3.large"
vpc_id        = "vpc-12345678"
subnet_id     = "subnet-abcdef12"
environment   = "prod"

- test.tfvars (For Testing Environment)
instance_type = "t2.micro"
vpc_id        = "vpc-87654321"
subnet_id     = "subnet-fedcba21"
environment   = "test"

{
  "resources": [
    {
      "type": "aws_instance",
      "name": "app",
      "instances": [
        {
          "attributes": {
            "ami": "ami-0c55b159cbfafe1f0",
            "instance_type": "t2.micro",
            "subnet_id": "subnet-fedcba21",
            "tags": {
              "Name": "test-app"
            }
          }
        }
      ]
    }
  ]
}
# How auto.tfvars Works
Any file ending in *.auto.tfvars is automatically loaded when running terraform apply.
No need to specify -var-file manually.
If -var-file is used, Terraform ignores *.auto.tfvars and only uses the explicitly mentioned variable file.
Example Without -var-file (auto.tfvars is used)

**Loading Priority (highest to lowest):**
1. Command line flags (`-var` and `-var-file`)
2. `*.auto.tfvars` / `*.auto.tfvars.json` (alphabetical order)
3. `terraform.tfvars.json`
4. `terraform.tfvars`
5. Environment variables (`TF_VAR_name`)
6. Default values in variable declarations

Default values → Stored in auto.tfvars (automatically loaded if no -var-file is specified).
Environment-specific values → Stored in prod.tfvars, test.tfvars, etc. (must be specified explicitly using -var-file).

## Advanced Provider Configuration

### 21. How do you configure multiple provider instances in Terraform?

**Answer:**
Multiple provider configurations allow you to manage resources across different regions, accounts, or with different settings in the same Terraform configuration.

# Default provider (us-east-1)
provider "aws" {
  region = "us-east-1"
}

# Additional provider configuration (us-west-2)
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

# Instance using the default provider (us-east-1)
resource "aws_instance" "east_instance" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

# Instance using the aliased provider (us-west-2)
resource "aws_instance" "west_instance" {
  provider      = aws.west  # Specifies the "west" alias
  ami           = "ami-0892d3c7ee96c37be"
  instance_type = "t2.micro"
}


**Example with multiple AWS accounts:**
```hcl
provider "aws" {
  alias   = "production"
  region  = "us-east-1"
  profile = "production"
}

provider "aws" {
  alias   = "development"
  region  = "us-east-1"
  profile = "development"
}
```

### 22. How do you implement cross-region and cross-account resource management in Terraform?

**Answer:**
Managing resources across multiple regions or accounts requires careful provider configuration and may involve assuming roles or using different authentication methods.

Cross-Region Management:
# Define providers for different regions
provider "aws" {
  region = "us-east-1"
  alias  = "east"
}

provider "aws" {
  region = "us-west-2"
  alias  = "west"
}

# Create resources in different regions
resource "aws_vpc" "east_vpc" {
  provider = aws.east
  cidr_block = "10.0.0.0/16"
}

resource "aws_vpc" "west_vpc" {
  provider = aws.west
  cidr_block = "10.1.0.0/16"
}

# Cross-region resource dependencies
resource "aws_ec2_transit_gateway_peering_attachment" "example" {
  provider                = aws.east
  peer_region             = "us-west-2"
  peer_transit_gateway_id = aws_ec2_transit_gateway.west.id
  transit_gateway_id      = aws_ec2_transit_gateway.east.id
}


--------------------
---------------------

### 23. Explain how provider alias works in module instantiation.

Terraform Multi-Region Deployment with Modules
This document outlines the setup for deploying Terraform modules across multiple AWS regions. It covers how to configure providers, call modules, and specify different regions for resources such as EC2 and S3.

Repository Structure
Repo 1 (Modules Repository)
Contains Terraform modules:
EC2 (for launching EC2 instances)
S3 (for creating S3 buckets)

Repo 2 (Infrastructure Repository)
Defines Terraform providers.

Calls modules from Repo 1.
Provider Configuration (Repo 2)

In Repo 2, define providers in providers.tf:
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-1"
}

To deploy EC2 in the default region, define the module in main.tf without specifying a provider:
module "ec2" {
  source = "path_to_repo1/modules/ec2"
}

Deploying EC2 in us-west-1
To deploy EC2 in us-west-1, explicitly pass the provider alias:
module "ec2_west" {
  source    = "path_to_repo1/modules/ec2"
  providers = {
    aws = aws.west
  }
}

To deploy an S3 bucket in us-west-1, pass the provider alias:
module "s3" {
  source    = "path_to_repo1/modules/s3"
  providers = {
    aws = aws.west
  }
}

If you need to deploy EC2 in the default region and S3 in us-west-1 within the same configuration file:
module "ec2" {
  source = "path_to_repo1/modules/ec2"
}
module "s3" {
  source    = "path_to_repo1/modules/s3"
  providers = {
    aws = aws.west
  }
}

- Summary
EC2 in us-east-1: No provider override needed.
EC2 in us-west-1: Specify provider alias (aws.west).
S3 in us-west-1: Specify provider alias (aws.west).
EC2 and S3 in separate locations: Define both modules in main.tf, specifying aws.west only for S3.

## Terraform State Operations  ------

### 24. How would you modify a resource that has drifted from the Terraform configuration?
Scenario: EC2 and S3 Drift Detection & Resolution

We assume that:
You initially deployed 10 EC2 instances and 2 S3 buckets using Terraform.
A developer manually created one extra EC2 instance and one extra S3 bucket outside of Terraform.

You need to detect, import, and sync these manually created resources.

Step 1: Detect Drift Using terraform plan
Run:

terraform plan
Expected Output:

No changes. Your infrastructure matches the configuration.
Note: 1 additional EC2 instance found in AWS that is not managed by Terraform:
- Instance ID: i-0abcdef1234567890

Note: 1 additional S3 bucket found in AWS that is not managed by Terraform:
- Bucket Name: my-manual-bucket
👉 This confirms that an extra EC2 instance and S3 bucket exist in AWS but are not managed by Terraform.

Step 2: Import Manually Created Resources into Terraform
Import the EC2 Instance

terraform import aws_instance.manual_instance i-0abcdef1234567890
Expected Output:
Import successful!
aws_instance.manual_instance:
  ID: i-0abcdef1234567890
  AMI: ami-0c55b159cbfafe1f0
  Instance Type: t3.medium
  ...

👉 The EC2 instance is now tracked in Terraform’s state file but is not yet in main.tf.
Import the S3 Bucket
Run:
terraform import aws_s3_bucket.manual_bucket my-manual-bucket
Expected Output:

Import successful!
aws_s3_bucket.manual_bucket:
  Bucket: my-manual-bucket
  Region: us-east-1
  ...
👉 The S3 bucket is now tracked in Terraform’s state file but is not yet in main.tf.

Step 3: Verify the Imported Resources
Run:
terraform show
Expected Output (Snippet for EC2 and S3):

# aws_instance.manual_instance:
resource "aws_instance" "manual_instance" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
  id            = "i-0abcdef1234567890"
}

# aws_s3_bucket.manual_bucket:
resource "aws_s3_bucket" "manual_bucket" {
  bucket = "my-manual-bucket"
}
👉 Terraform now recognizes these resources in the state file, but they are missing from main.tf.

Step 4: Update main.tf to Include the Imported Resources
Modify your Terraform configuration file (main.tf) to include these resources:
resource "aws_instance" "manual_instance" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
}

resource "aws_s3_bucket" "manual_bucket" {
  bucket = "my-manual-bucket"
}

✅ Now, your main.tf file matches the manually created infrastructure.
Step 5: Apply to Fully Sync Terraform with AWS
Run:terraform apply -auto-approve
Expected Output:
No changes. Your infrastructure matches the configuration.
👉 Terraform now fully manages the extra EC2 instance and S3 bucket. Everything is in sync!


### 25. How would you rename a resource in Terraform without destroying and recreating it?
- terraform state mv aws_instance.old_name aws_instance.new_name
✅ This ensures Terraform keeps track of the resource under the new name without thinking it's deleted.

- Then, manually update main.tf to reflect the new name:
resource "aws_instance" "new_name" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
}

- After that, run terraform plan to confirm everything is in sync:
terraform plan
✅ If done correctly, Terraform should show no changes instead of trying to destroy/recreate anything.

- Finally, apply the changes:
terraform apply -auto-approve

### 26. Explain the purpose of the `terraform state` commands and provide examples.

**Answer:**
The `terraform state` command provides subcommands for advanced state management. These operations modify the state file directly and should be used with caution.

**Key Subcommands:**

1. **list** - Shows all resources in the state:
   ```bash
   terraform state list
   terraform state list aws_instance.*  # Filter by resource type
   ```

2. **show** - Shows detailed information about a specific resource:
   ```bash
   terraform state show aws_instance.example
   ```

3. **mv** - Moves an item in state (rename or move between modules):
   ```bash
   # Rename a resource
   terraform state mv aws_instance.old aws_instance.new
   
   # Move a resource into a module
   terraform state mv aws_instance.example module.instances.aws_instance.example
   ```

4. **rm** - Removes items from state (without destroying resources):
   ```bash
   terraform state rm aws_instance.example
   ```

**Common Use Cases:**

1. **Refactoring Terraform Code**:
   ```bash
   # Restructuring resources into modules
   terraform state mv aws_vpc.main module.networking.aws_vpc.main
   terraform state mv aws_subnet.public[*] module.networking.aws_subnet.public[*]
   ```

2. **Handling Resource Deletion Without Recreation**:
   ```bash
   # Remove resource from state but keep it in infrastructure
   terraform state rm aws_iam_policy.too_permissive
   
   # Create new resource with desired configuration
   # Edit terraform code to add new resource
   terraform apply
   ```

3. **Recovering from State Corruption**:
   ```bash
   # Export current state
   terraform state pull > terraform.tfstate.backup
   
   # Edit the backup file to fix issues
   # Push fixed state
   terraform state push terraform.tfstate.backup
   ```


