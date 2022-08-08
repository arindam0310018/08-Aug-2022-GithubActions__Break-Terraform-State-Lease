# BREAK TERRAFORM STATE LEASE USING GITHUB ACTIONS

Greetings to my fellow Technology Advocates and Specialists.

This Blog post is a follow-up to my previous post - [Break Terraform State Lease Using Azure DevOps](https://dev.to/arindam0310018/break-terraform-state-lease-using-azure-devops-2fnj)

In this Session, I will demonstrate how to Break Terraform State Lease Using Github Actions.

| __AUTOMATION OBJECTIVE:-__ |
| --------- |
| Validate If Resource Group Exists. |
| Validate If Storage Account Exists. |
| Validate If Storage Account Container Exists. |
| Validate If Terraform State File Exists in the Specified Storage Account Container. |
| If any One of the above validation __DOES NOT PASS__, Pipeline will Fail immediately. |
| If All of the above validation is __SUCCESSFUL__, Pipeline will then check the Terraform Blob State. |
| If Terraform Blob State is == AVAILABLE, Pipeline fails with Exit Code 1. |
| If Terraform Blob State is == LEASED, Pipeline executes successfully by breaking the Terraform State Lease. |
| If Terraform Blob State is == BROKEN, Pipeline Still executes successfully without altering the present state. |

| __IMPORTANT TO NOTE:-__ |
| --------- |
| There is No way to find the Blob Lease State before executing the __az storage blob lease break__ command. |
| Refer the Link for more Information: [az cli blob lease break](https://docs.microsoft.com/en-us/cli/azure/storage/blob/lease?view=azure-cli-latest#az-storage-blob-lease-break) |

| __REQUIREMENTS:-__ |
| --------- |

1. Azure Subscription.
2. Service Principal with Required RBAC ( __Contributor__) applied on Subscription or Resource Group(s).
3. GitHub Repository with Mandatory __Repository Secrets__ and Optionally __Environments__ Configured. 
4. Microsoft DevLabs Terraform Extension Installed in Local System (VS Code Extension).

| __GITHUB REPOSITORY CONFIGURATION:-__ |
| --------- |
| __Repository Secrets__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1xsagkilaxyho79qeqfb.JPG) |
| Refer the MS Documentation [Authenticate from Azure to Github](https://docs.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-cli%2Clinux#use-the-azure-login-action-with-a-service-principal-secret) for more details. |
| __Environments__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5swa57xn0615w2akjrfu.JPG) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/24yl9rosvfflylf3nwae.jpg) |

| __REPOSITORY SECRETS FORMAT:-__ |
| --------- |
| Below is how you configure Repository Secret named __AZURE_CREDENTIALS__ |

```
{
"clientId": "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
"clientSecret": "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
"subscriptionId": "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
"tenantId": "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
}
```
| __HOW DOES MY CODE PLACEHOLDER LOOKS LIKE:-__ |
| --------- |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0ksgvtame4bs9668spik.JPG) |

| PIPELINE CODE SNIPPET:- | 
| --------- |

| GITHUB ACTIONS YAML WORKFLOW (gh-actions-tf-break-lease-v1.0.yml):- | 
| --------- |

```
name: 'Break Terraform Lease'

on: [workflow_dispatch]

env:
  SUBSCRIPTIONID: '210e66cb-55cf-424e-8daa-6cad804ab604' 
  RGNAME: 'tfpipeline-rg'
  STORAGEACCOUNTNAME: 'tfpipelinesa'
  STORAGEACCOUNTCONTAINERNAME: 'terraform'
  TFSTATEFILENAME: 'TF-LEASE/BreakLease.tfstate'
   
   
jobs:

  build-and-deploy:
    runs-on: windows-latest
    environment: dev
    steps:

    - name: 'Azure CLI Login'
      uses: azure/login@v1
      with:
        creds: '${{ secrets.AZURE_CREDENTIALS }}'

    - name: 'Break TF State'
      run: |
          az --version
          az account set --subscription ${{ env.SUBSCRIPTIONID }}
          az account show  
          $i = az group exists -n ${{ env.RGNAME }}
            if ($i -eq "true") {
              echo "#####################################################"
              echo "Resource Group ${{ env.RGNAME }} exists!!!"
              echo "#####################################################"
              $j = az storage account check-name --name ${{ env.STORAGEACCOUNTNAME }} --query "reason" --out tsv
                if ($j -eq "AlreadyExists") {
                  echo "#####################################################"
                  echo "Storage Account ${{ env.STORAGEACCOUNTNAME }} exists!!!"
                  echo "#####################################################"  
                  $k = az storage container exists --account-name ${{ env.STORAGEACCOUNTNAME }} --account-key $(az storage account keys list -g ${{ env.RGNAME }} -n ${{ env.STORAGEACCOUNTNAME }} --query [0].value -o tsv) --name ${{ env.STORAGEACCOUNTCONTAINERNAME }} --query "exists" --out tsv 
                    if ($k -eq "true") {
                      echo "#####################################################"
                      echo "Storage Account Container ${{ env.STORAGEACCOUNTCONTAINERNAME }} exists!!!"
                      echo "#####################################################"  
                      $l = az storage blob exists --account-name ${{ env.STORAGEACCOUNTNAME }} --account-key $(az storage account keys list -g ${{ env.RGNAME }} -n ${{ env.STORAGEACCOUNTNAME }} --query [0].value -o tsv) --container-name ${{ env.STORAGEACCOUNTCONTAINERNAME }} --name ${{ env.TFSTATEFILENAME }} --query "exists" --out tsv  
                        if ($l -eq "true") {
                          echo "#####################################################"
                          echo "Terraform State Blob ${{ env.TFSTATEFILENAME }} exists!!!"
                          echo "#####################################################"
                          az storage blob lease break --account-name ${{ env.STORAGEACCOUNTNAME }} --account-key $(az storage account keys list -g ${{ env.RGNAME }} -n ${{ env.STORAGEACCOUNTNAME }} --query [0].value -o tsv) --container-name ${{ env.STORAGEACCOUNTCONTAINERNAME }} --blob-name ${{ env.TFSTATEFILENAME }}                           
                          }
                        else {
                          echo "#####################################################"
                          echo "Terraform State Blob ${{ env.TFSTATEFILENAME }} DOES NOT EXISTS in ${{ env.STORAGEACCOUNTCONTAINERNAME }}"
                          echo "#####################################################"
                          exit 1
                          }
                    }
                    else {
                      echo "#####################################################"
                      echo "Storage Account Container ${{ env.STORAGEACCOUNTCONTAINERNAME }} DOES NOT EXISTS in STORAGE ACCOUNT ${{ env.STORAGEACCOUNTNAME }}!!!"
                      echo "#####################################################"
                      exit 1
                    }
                }
                else {
                  echo "#####################################################"
                  echo "Storage Account ${{ env.STORAGEACCOUNTNAME }} DOES NOT EXISTS!!!"
                  echo "#####################################################"
                  exit 1
                }              
            }
            else {
              echo "#####################################################"
              echo "Resource Group ${{ env.RGNAME }} DOES NOT EXISTS!!!"
              echo "#####################################################"
              exit 1
            }

```

Now, let me explain each part of GitHub Actions YAML Workflow for better understanding.

| __PART 1:-__ |
| --------- |

| __BELOW FOLLOWS WORKFLOW TRIGGER CODE SNIPPET:-__ |
| --------- |

```
on: [workflow_dispatch]

```
This indicates that GitHub Actions Workflow needs to be triggered manually.


| __PART 2:-__ |
| --------- |

| __BELOW FOLLOWS WORKFLOW VARIABLES CODE SNIPPET:-__ |
| --------- |

```
env:
  SUBSCRIPTIONID: '210e66cb-55cf-424e-8daa-6cad804ab604' 
  RGNAME: 'tfpipeline-rg'
  STORAGEACCOUNTNAME: 'tfpipelinesa'
  STORAGEACCOUNTCONTAINERNAME: 'terraform'
  TFSTATEFILENAME: 'TF-LEASE/BreakLease.tfstate'

```

| __NOTE:-__ |
| --------- |
| Please feel free to change the values of the variables. |
| The entire workflow is build using variables. No Values are Hardcoded. |
| In GitHub Actions, the variables are designed to work in such a way that the __Declared Variables are Mapped to Environmental Variables__. |


| __PART 3:-__ |
| --------- |

| __BELOW FOLLOWS RUNNER AND ENVIRONMENT CODE SNIPPET:-__ |
| --------- |

```
runs-on: windows-latest
environment: dev
```

| __NOTE:-__ |
| --------- |
| __RUNNER__ in GitHub Actions = __BUILD AGENT__ in Azure DevOps. |
| ENVIRONMENT = Workflow Environment where Approval Gate is Configured. |


| __PART 4:-__ |
| --------- |

| __BELOW FOLLOWS AZURE CLI LOGIN CODE SNIPPET USING GITHUB ACTIONS FOR AZURE CLI:-__ |
| --------- |

```
- name: 'Azure CLI Login'
      uses: azure/login@v1
      with:
        creds: '${{ secrets.AZURE_CREDENTIALS }}'
```

| __NOTE:-__ |
| --------- |
| Refer [GitHub Actions For Azure CLI](https://github.com/marketplace/actions/azure-cli-action) for more details. |

| __PART 5:-__ |
| --------- |

| __BELOW FOLLOWS BREAK TERRAFORM STATE LEASE CODE SNIPPET :-__ |
| --------- |

```
- name: 'Break TF State'
      run: |
          az --version
          az account set --subscription ${{ env.SUBSCRIPTIONID }}
          az account show  
          $i = az group exists -n ${{ env.RGNAME }}
            if ($i -eq "true") {
              echo "#####################################################"
              echo "Resource Group ${{ env.RGNAME }} exists!!!"
              echo "#####################################################"
              $j = az storage account check-name --name ${{ env.STORAGEACCOUNTNAME }} --query "reason" --out tsv
                if ($j -eq "AlreadyExists") {
                  echo "#####################################################"
                  echo "Storage Account ${{ env.STORAGEACCOUNTNAME }} exists!!!"
                  echo "#####################################################"  
                  $k = az storage container exists --account-name ${{ env.STORAGEACCOUNTNAME }} --account-key $(az storage account keys list -g ${{ env.RGNAME }} -n ${{ env.STORAGEACCOUNTNAME }} --query [0].value -o tsv) --name ${{ env.STORAGEACCOUNTCONTAINERNAME }} --query "exists" --out tsv 
                    if ($k -eq "true") {
                      echo "#####################################################"
                      echo "Storage Account Container ${{ env.STORAGEACCOUNTCONTAINERNAME }} exists!!!"
                      echo "#####################################################"  
                      $l = az storage blob exists --account-name ${{ env.STORAGEACCOUNTNAME }} --account-key $(az storage account keys list -g ${{ env.RGNAME }} -n ${{ env.STORAGEACCOUNTNAME }} --query [0].value -o tsv) --container-name ${{ env.STORAGEACCOUNTCONTAINERNAME }} --name ${{ env.TFSTATEFILENAME }} --query "exists" --out tsv  
                        if ($l -eq "true") {
                          echo "#####################################################"
                          echo "Terraform State Blob ${{ env.TFSTATEFILENAME }} exists!!!"
                          echo "#####################################################"
                          az storage blob lease break --account-name ${{ env.STORAGEACCOUNTNAME }} --account-key $(az storage account keys list -g ${{ env.RGNAME }} -n ${{ env.STORAGEACCOUNTNAME }} --query [0].value -o tsv) --container-name ${{ env.STORAGEACCOUNTCONTAINERNAME }} --blob-name ${{ env.TFSTATEFILENAME }}                           
                          }
                        else {
                          echo "#####################################################"
                          echo "Terraform State Blob ${{ env.TFSTATEFILENAME }} DOES NOT EXISTS in ${{ env.STORAGEACCOUNTCONTAINERNAME }}"
                          echo "#####################################################"
                          exit 1
                          }
                    }
                    else {
                      echo "#####################################################"
                      echo "Storage Account Container ${{ env.STORAGEACCOUNTCONTAINERNAME }} DOES NOT EXISTS in STORAGE ACCOUNT ${{ env.STORAGEACCOUNTNAME }}!!!"
                      echo "#####################################################"
                      exit 1
                    }
                }
                else {
                  echo "#####################################################"
                  echo "Storage Account ${{ env.STORAGEACCOUNTNAME }} DOES NOT EXISTS!!!"
                  echo "#####################################################"
                  exit 1
                }              
            }
            else {
              echo "#####################################################"
              echo "Resource Group ${{ env.RGNAME }} DOES NOT EXISTS!!!"
              echo "#####################################################"
              exit 1
            }

```

| __NOTE:-__ |
| --------- |
| __Az CLI__ commands are used with __RUN__ action, without the need of __GitHub Actions for Azure CLI (azure/login@v1)__ |


| __OBJECTIVE OF TERRAFORM CODE SNIPPET:-__ |
| --------- |
| Create a Resource Group and User Assigned System Managed Identity. |
| The Purpose of the Terraform Code Snippet is to Reproduce the Issue, by Locking Terraform State File . |


| __TERRAFORM (main.tf):-__ |
| --------- |

```
terraform {
  required_version = ">= 1.2.3"

   backend "azurerm" {
    resource_group_name  = "tfpipeline-rg"
    storage_account_name = "tfpipelinesa"
    container_name       = "terraform"
    key                  = "TF-LEASE/BreakLease.tfstate"
  }
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.2"
    }   
  }
}
provider "azurerm" {
  features {}
  skip_provider_registration = true
}

```

| __TERRAFORM (usrmid.tf):-__ |
| --------- |

```
## Azure Resource Group:-
resource "azurerm_resource_group" "rg" {
 name     = var.rg-name
 location = var.rg-location
}

## Azure User Assigned Managed Identities:-
resource "azurerm_user_assigned_identity" "az-usr-mid" {
  
 name                = var.usr-mid-name
 resource_group_name = azurerm_resource_group.rg.name
 location            = azurerm_resource_group.rg.location
  
 depends_on          = [azurerm_resource_group.rg]
 }

```

| __TERRAFORM (variables.tf):-__ |
| --------- |

```
variable "rg-name" {
  type        = string
  description = "Name of the Resource Group"
}

variable "rg-location" {
  type        = string
  description = "Resource Group Location"
}

variable "usr-mid-name" {
  type        = string
  description = "Name of the User Assigned Managed Identity"
}

```

| __TERRAFORM (usrmid.tfvars):-__ |
| --------- |

```
rg-name         = "AMTest100"
rg-location     = "West Europe"
usr-mid-name    = "AMUSRMID100"

```

| __HOW TO LOCK TERRAFORM STATE FILE:-__ |
| --------- |

Run the Terraform Apply Command manually in your local System as mentioned below:-

```
terraform apply --var-file="usrmid.tfvars"

```

When Prompted for "Yes", Press __Control + C__ to terminate the Execution.

| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/41fqa716hcaxk1x3kif4.png) |
| --------- |
 
Next time, when you Re-Execute the Command, It will Inform the User that Terraform State File is in Locked State. 

| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wdqqavufyu0qsr7xzn73.jpg) |
| --------- |


__NOW ITS TIME TO TEST !!!...__

| __TEST CASES:-__ |
| --------- |

| __TEST CASE #1: TERRAFORM BLOB STATE == AVAILABLE:-__ |
| --------- |
| __DESIRED OUTPUT: PIPELINE WILL FAIL WITH EXIT CODE 1.__ |
| __TERRAFORM BLOB STATE:-__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/n64zduldwcnrhxblnnqa.JPG) |
| WORKFLOW RUN:- |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zjpv3nv8lkbd0goaorh1.png) |
| __TEST CASE #2: TERRAFORM BLOB STATE == LEASED:-__ |
| __DESIRED OUTPUT: PIPELINE EXECUTES SUCCESSFULLY BREAKING TERRAFORM BLOB STATE LEASE.__ |
| __TERRAFORM BLOB STATE:-__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fwsnzpi678gpkdmokyuf.JPG) |
| __WORKFLOW RUN:-__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ba17rxbzb3eq9lu1oxws.png) |
| __TEST CASE #3: TERRAFORM BLOB STATE == BROKEN:-__ |
| __DESIRED OUTPUT: PIPELINE EXECUTES SUCCESSFULLY WITHOUT ANY ALTERATIONS TO THE CURRENT STATE.__ |
| __TERRAFORM BLOB STATE:-__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/al5omj2izkwkph3n26f8.jpg) |
| __WORKFLOW RUN:-__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/yp70joh4qs5oihgq5mxu.png) |

| __OVERALL GITHUB ACTIONS WORKFLOW RUNS:-__ |
| --------- |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/q0isbtbcg7z7zpx302ta.JPG) |


Hope You Enjoyed the Session!!!

Stay Safe | Keep Learning | Spread Knowledge
