# Managing infrastructure as code with Terraform, Cloud Build, and GitOps

This is the repo for the [Managing infrastructure as code with Terraform, Cloud Build, and GitOps](https://cloud.google.com/solutions/managing-infrastructure-as-code) tutorial. This tutorial explains how to manage infrastructure as code with Terraform and Cloud Build using the popular GitOps methodology. 

## Configuring your **dev** environment

Just for demostration, this step will:
 1. Configure an apache2 http server on network '**dev**' and subnet '**dev**-subnet-01'
 2. Open port 80 on firewall for this http server 

```bash
cd ../environments/dev
terraform init
terraform plan
terraform apply
terraform destroy
```

## Promoting your environment to **production**

Once you have tested your app (in this example an apache2 http server), you can promote your configuration to prodution. This step will:
 1. Configure an apache2 http server on network '**prod**' and subnet '**prod**-subnet-01'
 2. Open port 80 on firewall for this http server 

```bash
cd ../prod
terraform init
terraform plan
terraform apply
terraform destroy
```
---

## Experiences Following [Managing infrastructure as code with Terraform, Cloud Build, and GitOps](https://cloud.google.com/solutions/managing-infrastructure-as-code)

### Automating TF Cluster with Cloud Build
1. CloudBuild Service Account Requires roles/editor but I didn't have perm: `resourcemanager.projects.setIamPolicy`
    * I got project `owner` role and was able to give cloud build SA `roles/editor`
```
    $ PROJECT_ID=$(gcloud config get-value project)
    $ echo $PROJECT_ID 
    mms-sandbox-gitops
    
    $ CLOUDBUILD_SA="$(gcloud projects describe $PROJECT_ID \
        --format 'value(projectNumber)')@cloudbuild.gserviceaccount.com"
    $ echo $CLOUDBUILD_SA 
    666664191685@cloudbuild.gserviceaccount.com
    
    $ gcloud projects add-iam-policy-binding $PROJECT_ID \
         --member serviceAccount:$CLOUDBUILD_SA --role roles/editor
```
1. How can we automate GCP project creation, TF SA creation and giving TF (or tool executing TF, i.e., CloudBuild) project `editor` role?
    * Can bootstrap with a "bootstrap" script running `gcloud` commands.
1. How to `terraform destroy` infra when it's only in automation?
    * Either need to:
        1. Delete the whole project from the IAM admin page
            * permission issues
            * project reuse issues
        1. Init TF with remote state from bucket and `terraform destroy`?
            * need to be running same TF version locally as defined in modules or use [tfswitch (Terraform Switcher)](https://github.com/warrensbox/terraform-switcher)
            * need to be able to authenticate to remote state bucket from local machine or you'll see: `Error configuring the backend "gcs":... google: could not find default credentials.`
            * See [Terraform GCP Provider full reference](https://www.terraform.io/docs/providers/google/provider_reference.html#full-reference)
                * Downloading JSON key, setting path in `GOOGLE_APPLICATION_CREDENTIALS` env var and adding `credentials` to `provider "google"` works locally but **breaks cloud build automation**.  This example works to run locally:
```
                    locals {
                        "env" = "dev"
                        "GOOGLE_APPLICATION_CREDENTIALS" = "/Users/cunninj0/.config/gcloud/mm-sandbox-gitops.json"
                    }

                    provider "google" {
                        credentials = "${local.GOOGLE_APPLICATION_CREDENTIALS}"
                        project = "${var.project}"
                    }
```

* **TODO**: Experiment how to `terraform destroy` without deleting whole project.
    * Alter `cloudbuild.yaml` to add some sort of "flag" to trigger `destroy`.  Note, since we don't have direct access to the env, it would have to be something committed on the file system?
    * Just execute `terraform destroy` from the local cmd line. Requires init'ing with `GOOGLE_APPLICATION_CREDENTIALS` as above.  May also require matching local terraform version with cloud build.

## Observations
1. Looks like we'd have to mirror the repository into Cloud Source Repo if using BitBucket.
    * See [Cloud Build Additional Information](https://cloud.google.com/source-repositories/docs/integrating-with-cloud-build#additional_information)
1. Should configure src repo to only allow PR merge to default branch and have build set up to only run `terraform apply` on default branch.  `terraform init/plan` on other branches. (GitOps)
1. When updating cloud build config (cloudbuild.yaml) to upgrade Terraform version, make sure versions.tf in modules are updated too.
    * Also make sure provider accommodates the upgraded TF version.
1. Sometimes TF plan does not find breaking changes (e.g., change zone for http server, forgot to change region for VPC). This was not caught by init/plan, was merged and failed.
    * It's possible a savvy reviewer could spot an inconsistency in the plan output but, because of its verbosity, it's unlikely. 
1. No easy way to recover resources created in the project (`terraform destroy`) without deleting the whole project.
1. It seems a bit odd in [Managing infrastructure as code with Terraform, Cloud Build, and GitOps](https://cloud.google.com/solutions/managing-infrastructure-as-code) to have _both_ a `git branch` for an environment _and_ a directory (i.e., `environments/dev`).  Seems a little convoluted and repeditive.  Looks like only network differences between envs, which makes sense.  Maybe it's necessary for more complex systems but environments should be as close as possible and, hopefully, much can be reused for both (all). 
1. Destroying the `dev` environment via local `terraform destroy` _did not_ remove the `env/dev/` folder in the shared `mms-sandbox-gitops-tfstate` bucket.
    * Terraform is as much a "cultural" shift as it is automation.  It seems that probably _all_ changes to the infra should be done through GitOps automation - probably shouldn't run terraform locally.
