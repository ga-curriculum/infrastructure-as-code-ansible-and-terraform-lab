<h1>
  <span class="headline">Infrastructure As Code: Ansible & Terraform Lab</span>
  <span class="subhead">Exercise</span>
</h1>

## Creating an AWS RDS service with Terraform

In the previous lesson, you learned the fundamentals of using Terraform by creating an AWS DynamoDB instance. In this lab, you will gain additional practice by creating an AWS RDS (Relational Database Service) instance. You do not need to have any prior experience with RDS, this lab will simply give you an opportunity to experiment with its setup using Terraform.

## Step 1: Set up the Terraform manifest

1. **Open the file** `rds.tf` created during setup. You can locate this file in the `terraform-lab-01` directory and open it in VS Code:

   ```bash
   code rds.tf
   ```

2. **Add the Terraform and Provider blocks**:

   Copy and paste the following into the `rds.tf` file:

   ```hcl
    terraform {
      required_providers {
        aws = {
          source = "hashicorp/aws"
          version = "~> 4.16"
        }
      }

      required_version = ">= 1.2.0"
    }

    provider "aws" {
      region = "us-east-1"
    }
   ```

3. **Save the file**.

## Step 2: Initialize Terraform

1. **Run the initialization command** in the terminal:

   ```bash
   terraform init
   ```

   The output should look similar to this:

   ```text
   Initializing the backend...

   Initializing provider plugins...

   - Finding hashicorp/aws versions matching "~> 4.16"...
   - Installing hashicorp/aws v4.17.0...
   - Installed hashicorp/aws v4.17.0 (signed by HashiCorp)

   Terraform has been successfully initialized!
   ```

2. **If you encounter errors**, ask the instructor for assistance. Ensure this step is successful before moving forward.

## Step 3: Add AWS RDS resources

1. **Update the `rds.tf` file** by adding the following resource blocks at the end of the file:

   ```hcl
   resource "aws_rds_cluster" "modern-engineering-aurora-postgressql-cluster" {
     cluster_identifier      = "modern-engineering-aurora-postgresql-cluster"
     engine                  = "aurora-postgresql"
     engine_version          = "11.9"
     availability_zones      = ["us-east-1a", "us-east-1b", "us-east-1c"]
     master_username         = "myusername"
     master_password         = "The.5ecret?P4ssw0rd"
     database_name           = "mydatabase"
     backup_retention_period = 5
     preferred_backup_window = "07:00-09:00"
     apply_immediately       = true
     skip_final_snapshot     = true
   }

   resource "aws_rds_cluster_instance" "modern-engineering-aurora-postgressql-instances" {
     count              = 2
     identifier         = "modern-engineering-aurora-postgresql-instance-${count.index}"
     cluster_identifier = aws_rds_cluster.modern-engineering-aurora-postgressql-cluster.id
     instance_class     = "db.t3.medium"
     engine             = aws_rds_cluster.modern-engineering-aurora-postgressql-cluster.engine
     engine_version     = aws_rds_cluster.modern-engineering-aurora-postgressql-cluster.engine_version
   }

   output "db_host" {
     value       = aws_rds_cluster.modern-engineering-aurora-postgressql-cluster.endpoint
     description = "The endpoint of the RDS cluster"
   }
   ```

> Notice we added an `output` block at the bottom of this `rdd.tf` file. This will print the database endpoint (DB host) when you run `terraform apply`, making it easy for you to use in the Ansible section of the exercise.

2. **Replace the placeholder values**:

   ```text
   master_username         = "myusername"
   master_password         = "The.5ecret?P4ssw0rd"
   database_name           = "mydatabase"
   ```

- master_username: Replace "myusername" with a suitable username (ex: "admin").

- master_password: Replace "The.5ecret?P4ssw0rd" with a strong, unique password.

- database_name: Replace "mydatabase" with an appropriate name for your database (ex: "moviedb").

3. **Save the file before running.**

<blockquote class="attention">
  ⚠️ Important: Ensure you do not hard-code passwords in real world scenarios. Use environment variables or tools like AWS Secrets Manager to securely manage passwords. Do not commit sensitive information such as passwords to version control.
</blockquote>

## Step 4: Plan and apply the Terraform configuration

1. **Run the plan command** to preview the changes:

   ```bash
   terraform plan
   ```

   - If you encounter an error, rerun `terraform init` and then `terraform plan`.

2. **Apply the changes**:

   ```bash
   terraform apply
   ```

   - When prompted, type `yes` to confirm.
   - This process may take about 15 minutes.

3. **Verify the RDS service**:

   - Log in to the AWS Management Console.
   - Navigate to the RDS service and confirm the RDS instance has been created.

## Step 4: (Optional) Configure the database with Ansible

In this bonus section, you will use Ansible to further configure the RDS instance created in the previous steps. Specifically, you will connect to the RDS database and create a table called `movies`.

### Required installations for this exercise

1. `postgresql-client`: To enable interaction with PostgreSQL databases for the playbook.

   ```bash
   sudo apt-get install -y postgresql-client
   ```

2. **In your teminal**, create a new Ansible `playbook` file:

   ```bash
   touch configure_rds.yml
   ```

3. **Copy and Paste** the following content into your new `playbook`:

   ```yml
   ---
   - name: Configure AWS RDS Database
     hosts: localhost
     gather_facts: no

     vars:
       db_host: 'your-db-endpoint' # Update placeholder value
       db_name: 'mydatabase' # Update placeholder value
       db_user: 'myusername' # Update placeholder value
       db_password: 'The.5ecret?P4ssw0rd' # Update placeholder value
       db_port: '5432'

     tasks:
       - name: Create 'movies' table
         community.postgresql.postgresql_query:
           db: '{{ db_name }}'
           login_user: '{{ db_user }}'
           login_password: '{{ db_password }}'
           login_host: '{{ db_host }}'
           login_port: '{{ db_port }}'
           query: |
             CREATE TABLE IF NOT EXISTS movies (
                 id SERIAL PRIMARY KEY,
                 title VARCHAR(255) NOT NULL,
                 director VARCHAR(255),
                 release_year INT,
                 genre VARCHAR(100)
             );
         register: query_result

       - name: Display result
         debug:
           var: query_result
   ```

> You can obtain the variables for the Ansible configuration by referencing the Terraform output or by manually gathering the details from the AWS Management Console after the RDS instance is created.

3. **Run the playbook** to create the table:

   ```bash
   ansible-playbook configure_rds.yml
   ```

4. **Verify** the creation of the `movies` table in your AWS Management Console.

   Great Job!

## Step 5: Clean up resources

1. **Destroy the infrastructure** to clean up:

   ```bash
   terraform destroy --auto-approve
   ```

2. **Verify deletion**:

   - Wait approximately 15 minutes.
   - Check the AWS Management Console to confirm the RDS instance has been deleted.

## Summary

In this lab, you:

- Created an AWS RDS instance using Terraform.
- Practiced the steps to initialize, plan, apply, and destroy a Terraform configuration.
  - `terraform init`
  - `terraform plan`
  - `terraform apply`
  - `terraform destroy`
- Used Ansible to configure the database and create a table. (Bonus)
- Created a table named `movies` using Ansible tasks (Bonus)
