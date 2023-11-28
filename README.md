# Task 7: CI/CD

</br>

🎯 **Goal:** to create first CI/CD pipeline using GitHub Actions

</br>

>[!IMPORTANT]
>Config files describing resources commented out due to GCP infrastrucutre cleanup for task 8 - terraform deactivated </br>
>Below steps describe task that had been performed before cleanup

</br>

💪 **Task steps**

</br>

**Step 1** Create a Service Account for Terraform
  - Go to IAM in the GCP Console
  - Give it any name you like e.g. terraform and click "Create"
  - For the Role, choose "Project -> Editor", then click "Continue"
  - Skip granting additional users access, and click "Done"

</br> 


**Step 2** Generate the Key for created Service Account

  - In tab KEYS, click ADD KEY and choose JSON for the Key type

</br> 

**Step 3** Modify the downladed key content - remove all new line characters from the file

  - use vim to create a temp file:

</br> 

     vim temp_content_key.json

</br> 

  - paste the content of the downloaded key file
  - Press :
  - Type: %s;\\n; ;g and press enter
  - Press :
  - Save the file by typing wq!
  - download the file to your local drive (you can open Editor to do that)'

</br>

  ![image](https://github.com/IKRadwan/dareit-terraform/assets/146995869/16be5521-b3bd-4727-ae8f-eb1f14c44ab6)

</br>

>[!CAUTION]
> This step has been abandoned due to generated error within workflow on Terraform Init Step, on the further stage (**Step 12**):
></br>
> `Error: Failed to get existing workspaces: querying Cloud Storage failed: and private key should be a PEM or plain PKCS1 or PKCS8; parse error: asn1: structure error: tags don't match`
></br>
> Not modified key content updated to GitHab as a secret (**Step 7**) - it has not generated any error on the further stage

</br>

**Step 4** Create a bucket in UI in which the terraform state file will be stored
  - **do not** grant public access to this bucket

</br>

**Step 5** Create a new repository in Github named "dareit-terraform"

</br>

**Step 6** Provide the key content as a new secret in the created repository
  - Go to the Settings tab
  - Choose "Actions" under the Security section
  - Click the button "New repository secret"
  - Paste the content of the key file generated in **Step 2**
  - Name the secret "TF_GOOGLE_CREDENTIALS"

</br>

**Step 7** Download (clone) the repository to your local 

</br>

**Step 8** Create files in your local repository:

</br>

  - **backend.tf** (provide the name of the bucket you created in **Step 4**)

```
  terraform {  
  required_version = ">= 1.0.11"
  backend "gcs" {
    bucket = "YOUR_BUCKET_NAME_FOR_STATE_FILE"
    prefix = "dev"
  }
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "4.41.0"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "4.41.0"
    }
  }
}
```
</br>

- **provider.tf** (input the project id)
  
```
provider "google" {
  project = "YOU_GCP_PROJECT_ID"
  region  = "us-central1"
  zone    = "us-central1-c"
}
```
</br>

  - **main.tf**

```
resource "google_compute_instance" "dareit-vm-ci" {
  name         = "dareit-vm-tf-ci"
  machine_type = "e2-medium"
  zone         = "us-central1-a"

  tags = ["dareit"]

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
      labels = {
        managed_by_terraform = "true"
      }
    }
  }

  network_interface {
    network = "default"

    access_config {
      // Ephemeral public IP
    }
  }
}
```
</br>

**Step 9** Create the file with the definition of the pipeline: 
</br>
:point_right: the pipeline to run every time a new commit is pushed to the main branch in the repository 
</br>
:point_right: the pipeline has a few steps:
</br>
  
```
Checkout

Setup Terraform

Terraform Init

Terraform Formt

Terraform Plan

Terraform Apply

```

  - Create a file in your repository .github/workflows/terraform.yml </br>
📖 A few of the steps read the secret variable from the repository in order to be able to authenticate to GCP and be able to perform certain actions on the GCP resources
  ```
name: 'Terraform CI'

on:
  push:
    branches:
    - main

jobs:
  terraform:
    name: 'Terraform'
    runs-on: ubuntu-latest

    # Use the Bash shell regardless whether the GitHub Actions runner is ubuntu-latest, macos-latest, or windows-latest
    defaults:
      run:
        shell: bash

    steps:
    # Checkout the repository to the GitHub Actions runner
    - name: Checkout
      uses: actions/checkout@v2

    # Install the latest version of Terraform CLI
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v1


    # Initialize a new or existing Terraform working directory by creating initial files, loading any remote state, downloading modules, etc.
    - name: Terraform Init
      run: terraform init
      env:
        GOOGLE_CREDENTIALS: ${{ secrets.TF_GOOGLE_CREDENTIALS }}

    # Run terraform fmt to check whether the formatting of the files is correct
    - name: Terraform Format
      run: terraform fmt -check

    # Run terraform plan
    - name: Terraform Plan
      run: terraform plan
      env:
        GOOGLE_CREDENTIALS: ${{ secrets.TF_GOOGLE_CREDENTIALS }}

    # Run terraform apply
    - name: Terraform Apply
      run: terraform apply -auto-approve
      env:
        GOOGLE_CREDENTIALS: ${{ secrets.TF_GOOGLE_CREDENTIALS }}
```
</br>

![image](https://github.com/IKRadwan/dareit-terraform/assets/146995869/5042eb8d-e977-4102-8585-0b2741ac61f6)

</br>

**Step 10** Commit all files to the repo and push the change to the remote repository

</br>

**Step 11** Check the Actions tab in the repository on Github

  - see the workflow in there
    

![image](https://github.com/IKRadwan/dareit-terraform/assets/146995869/7d6b4899-a25b-4913-ad06-c08890557d17)

  - Click the workflow and see the details of it then, click on the Terraform Job

![image](https://github.com/IKRadwan/dareit-terraform/assets/146995869/a68eceff-b275-4a8f-bfe5-b757e0bdcdf2)

  - check a step with error and provide the appropriate resolution

    ![image](https://github.com/IKRadwan/dareit-terraform/assets/146995869/10a2fcd7-b328-4d98-87be-2d18e69877dc)
    
</br>

:point_right: To resolve the above error in Terraform Init, secret key has been updated on Github with no modified content (see **Step 3**) and all jobs in workflow re-run
</br>
:point_right: secret credentials block added to step Terraform Format:
```
env:
        GOOGLE_CREDENTIALS: ${{ secrets.TF_GOOGLE_CREDENTIALS }}
```
:lock: ❗ Delete key file from local drive

</br>

**Step 12** Once all steps are successfully completed, check the terraform state file on GCP (it should be created on a bucket created in **Step 4**)

</br> 

**Step 13** Modify the workflow so that the job will only be triggered whenever a new pull request is opened

</br> 

**Step 14** Create a branch called feat/add-bucket

</br> 

**Step 15** On feat/add-bucket branch in local repository, modify the code of the main.tf to add a new bucket 

</br> 

**Step 16** Commit the change and push to the repository on Github 
👉 no workflow run as per modified terraform.yml file

</br> 

**Step 17** Open a pull request 
👉workflow run

</br> 

📌As workflow failed on Terraform Format step, main file has been modified as per [Terraform guideline](https://developer.hashicorp.com/terraform/language/syntax/style)


**Step 18** Modify terraform.yml so the workflow is also triggered for pushes




