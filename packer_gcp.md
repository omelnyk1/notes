## Packer in GCP
# Intro
Packer is an open source tool for creating identical machine images for multiple platforms from a single source configuration.
It's lightweight, runs on every major operating system, and is highly performant, creating machine images for multiple platforms in parallel.
Packer does not replace configuration management like Chef or Puppet.
In fact, when building images, Packer is able to use tools like Chef or Puppet to install software onto the image.
A machine image is a single static unit that contains a pre-configured operating system and installed software which is used to quickly create new running machines.
Machine image formats change for each platform. Some examples include AMIs for EC2, VMDK/VMX files for VMware, OVF exports for VirtualBox, etc.
# Prerequires
To run packer and build machine images we should prepare GPC IAM service account and set proper roles.
This stage is described on [official documentation](https://www.packer.io/docs/builders/googlecompute#running-on-google-cloud).
Terraform snippet:
```tf
...
locals {
  packer_iam_roles = [
    "roles/iam.serviceAccountUser",
    "roles/compute.instanceAdmin.v1"
  ]
}

resource "google_service_account" "packer" {
  account_id   = "packer"
  display_name = "Packer Service Account"
  description  = "Packer Service Account https://www.packer.io/docs/builders/googlecompute.html#running-on-google-cloud"
}

resource "google_project_iam_binding" "packer" {
  count = length(local.packer_iam_roles)

  project = "test-project"
  role    = element(local.packer_iam_roles, count.index)

  members = [
    "serviceAccount:${google_service_account.packer.email}"
  ]
}
...
```
To run Packer outside of GCP we should create and save key for IAM service account:
```sh
gcloud iam service-accounts keys create gcloud-key.json --iam-account packer@test-project.iam.gserviceaccount.com
```
Set environmnet varibale to point to the path of the service account key:
```sh
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/gcloud-key.json
```
To install Packer please follow [instructions](https://www.packer.io/downloads).
# Configuration example
Here is an example of packer configuration file:
```json
{
  "variables": {
    "gcp_default_zone": "europe-west1-d",
    "source_image": "centos-8-v20201014",
    "git_commit": "{{ env `GIT_COMMIT` }}",
    "git_branch": "{{ env `GIT_BRANCH` }}",
    "build_number": "{{ env `BUILD_NUMBER` }}",
    "dest_dir": "/tmp"
  },
  "builders": [
    {
      "type": "googlecompute",
      "zone": "{{ user `gcp_default_zone` }}",
      "project_id": "test-project",
      "source_image": "{{ user `source_image` }}",
      "disk_size": 20,
      "metadata": {
        "enable-oslogin": "TRUE"
      },
      "use_os_login": true,
      "disable_default_service_account": true,
      "ssh_username": "packer",
      "image_family": "custom-rhel-image",
      "image_name": "custom-v{{isotime \"20060102t030405\"}}",
      "image_description": "Custom RHEL image",
      "image_labels": {
        "name": "rhel",
        "git_commit": "{{ user `git_commit` }}",
        "git_branch": "{{ user `git_branch` }}",
        "build_number": "{{ user `build_number` }}"
      }
    }
  ],
  "provisioners": [
    {
      "type": "file",
      "source": "files/",
      "destination": "{{ user `dest_dir` }}"
    },
    {
      "type": "shell",
      "inline": [
        "sudo sh {{ user `dest_dir` }}/setup/bootstrap.sh",
      ]
    }
  ]
}

```
In that example we will bake our custom image in GCP project `test-project`.

Temporary virtual machine will be provisioned in zone `europe-west1-d` (we can set it with user variable `gcp_default_zone`).
During baking process we don't have any interaction with other GCP services (CloudSQL, KMS, etc). That's why, according to Principle of Least Privilege (POLP), we set option `disable_default_service_account` - Packer will not assign default IAM serivce account on VM.
To link with CI tool and git commit all our images we will label using Jenkins environment varibales: `GIT_COMMIT`, `GIT_BRANCH`, `BUILD_NUMBER`.

Provisioning has 2 steps:
* copying scripts from local directory /files to remote /tmp
* running remove script /tmp/setup/bootstrap.sh

# References
* https://learn.hashicorp.com/tutorials/packer/getting-started-install
* https://www.packer.io/docs/builders/googlecompute#running-on-google-cloud
