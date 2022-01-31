# svc_on_instance

This is a simple app that installs a node service on a compute engine VM with Google Cloud Build + ansible.

## Requirements

- Create a CentOS 8 VM from compute engine console or CLI.
- Open 4000/tcp port at VPC network
- Open 4000/tcp port at VM using following cmd:
- Have access to a gcloud CLI with access to cloud account
  
```bash
sudo firewall-cmd --zone=public --permanent --add-port=4000/tcp
sudo firewall-cmd --reload
```


# Setup Automation

## Preperation steps

Obtain external ip and user of VM. User is generally same with gmail prefix on compute engine by default. 

Create a ssh key with following cmd:

```bash
ssh-keygen -t rsa -f ~/.ssh/id_rsa_gcp -C <user> -b 2048
```

Add public version of ssh key to google cloud metadata ssh keys section.

Add private key to Secret Manager with id_rsa_gcp name.

Create a service account to get access to Google Container Registry:

```bash
gcloud iam service-accounts create dockeruser
gcloud projects add-iam-policy-binding $PROJECT_ID --member "serviceAccount:dockeruser@$PROJECT_ID.iam.gserviceaccount.com" --role "roles/storage.admin"
gcloud iam service-accounts keys create keyfile.json --iam-account dockeruser@$PROJECT_ID.iam.gserviceaccount.com
gcloud auth activate-service-account dockeruser@$PROJECT_ID.iam.gserviceaccount.com --key-file=keyfile.json
```

Get access token for GCR. 

```bash
gcloud auth print-access-token
```

Add access token to Secret Manager with gcr_access_token name.

! This token has 60 minute lifetime before expire so add new version whenever 60 minutes pass from your last commit. Planning to improve this later.

## Setting up cloud build 

Right now we have following variables and secrets.

- VM External IP
- VM username
- id_rsa_gcp
- gcr_access_token


Login to cloud build page. 

From settings, enable Secret Manager service

From triggers, click create trigger.

Use following inputs:

- Name: give a name to trigger i.e.: svc-on-instance-main
- Source: connect this repository
- Branch: select a specific branch or leave as main
- Configuration: Select cloud build configuration file
- Advanced: Substitution variables: Add following two 
  - variable1: _ANSIBLE_HOST value1: VM External IP
  - variable2: _ANSIBLE_USER value2: VM username
- Save trigger


Now our pipeline is almost ready. Go to couldbuild.yaml and check secret versions are correct:

```
  secretManager:
  - versionName:   projects/$PROJECT_ID/secrets/id_rsa_gcp/versions/<version>
    env: 'RSAKEY'
  - versionName:   projects/$PROJECT_ID/secrets/gcr_access_token/versions/<version>
    env: 'GCRTOKEN'
```

Update if needed, and cloud build will automatically start otherwise click run on Triggers page to start.


## Pipeline workflow

Pipeline has 5 steps.

- Step 0: Create docker image for service
- Step 1: Push docker image to GCR 
- Step 2: Parse variables and update hosts.yml for ansible
- Step 3: Run service installation steps
- Step 4: Run service test steps


## Test workflow

- Step 1: Check if OS and OS-release are correct
- Step 2: Check if service is up and running with correct image version
- Step 3: Check if service port is listening
- Step 4: Check if service returns expected response internally
- Step 5: Check if service returns expected response externally

Test automatically runs as part of automation. To run test locally, configure hosts.yml accordingly and run following cmd:

```bash
ansible-playbook -i hosts.yml svc_tests.yml
```

## File structure 

```
$ tree
.
├── cloudbuild.yaml        // cloud build pipeline script
├── hosts.yml              // default hosts.yml for ansible
├── README.md              // README Documentation
├── svc_docker
│   ├── Dockerfile         // Dockerfile fore creating service container
│   ├── package.json       // Service dependencies file
│   ├── package-lock.json  // Service dependencies file
│   └── svc_main.js        // Service script file
├── svc_installer.yml      // Ansible playbook for service installation
└── svc_tests.yml          // Ansible playbook for service tests
```

# To-Dos

- Add a cron job creation step to pipeline for running tests periodically.
- Improve documentation and comments.
- Remove debug codes.  
- Improve ansible script structure.
- Find a better way to handle GCR access