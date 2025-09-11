# Notes - Setup for AlphaFold V2 Inference Pipelines on Vertex AI

## Pre-requisties

* Google Cloud project created and billing enabled
* user running these steps has `Owner` role on project

## Create GCE VM 

Oppen your Cloud Shell or local terminal and set the following environment variables. This will allow you to copy and paste the commands below without modification.

```bash
export PROJECT_ID="YOUIR-PROJECT-ID"
export ZONE="us-central1-a"
export INSTANCE_NAME="alphafold-data-prep-vm"
export MACHINE_TYPE="n2-standard-16" 
export DISK_SIZE_GB="5000"
export BUCKET_NAME="YOUIR-PROJECT-ID-alphafold-bucket"
export MOUNT_DIR="/mnt/disks/alphafold-data"
```

Create a VM instance with an attached persistent disk for the data. Using an **N2 machine type** is recommended for better network performance, which is crucial for large downloads and uploads.

```bash
gcloud compute instances create "${INSTANCE_NAME}" \
    --project="${PROJECT_ID}" \
    --zone="${ZONE}" \
    --machine-type="${MACHINE_TYPE}" \
    --network-interface=network-tier=PREMIUM \
    --boot-disk-size=50GB \
    --create-disk=auto-delete=no,boot=no,device-name=data-disk,mode=rw,size="${DISK_SIZE_GB}",type=pd-ssd \
    --scopes=https://www.googleapis.com/auth/cloud-platform 
```

**Note:** The scope `https://www.googleapis.com/auth/cloud-platform` grants the VM full access to Google Cloud services, including Cloud Storage.


## Create GCS Bucket 

Run the command below to create a bucket (only if it doesnt exist already)

```bash
gsutil mb -p "${PROJECT_ID}" "gs://${BUCKET_NAME}/"
```
Grant the service account the storage admin role on the bucket: 

```bash
export PROJECT_ID=$(gcloud config get-value project)
PROJECT_NUMBER=$(gcloud projects describe "${PROJECT_ID}" --format="value(projectNumber)")
export GCE_SA="${PROJECT_NUMBER}-compute@developer.gserviceaccount.com"
gsutil iam ch "serviceAccount:${GCE_SA}:roles/storage.admin" "gs://${BUCKET_NAME}"
```

## Configure GCE VM 

Next, SSH into your new VM to format and mount the attached large (TB) disk.

1. **SSH into the instance:**

Compute Engine > Click on "SSH" button of the newly created instance, run commands below. 

Or run following from local environment if already setup and familar with this part.

```bash
gcloud compute ssh "${INSTANCE_NAME}" --zone="${ZONE}" --project="${PROJECT_ID}"
```

Then run env vars again

```bash
export PROJECT_ID="YOUIR-PROJECT-ID"
export ZONE="us-central1-a"
export INSTANCE_NAME="alphafold-data-prep-vm"
export MACHINE_TYPE="n2-standard-16" 
export DISK_SIZE_GB="5000"
export BUCKET_NAME="YOUIR-PROJECT-ID-alphafold-bucket"
export MOUNT_DIR="/mnt/disks/alphafold-data"
```

confirm disks available for use

```sh
sudo lsblk
```

### Format Disk

1. **Format and mount the disk inside the VM:** The following commands should be run after you've connected via SSH.

```bash
# Unmount the data disk if it was auto-mounted by the OS (ignore errors if it's not mounted)
sudo umount /dev/sdb || true

# Format the attached data disk (it's /dev/sdb in this case)
sudo mkfs.ext4 -m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb

# Create the mount point directory
sudo mkdir -p "${MOUNT_DIR}"

# Mount the disk to that directory
sudo mount -o discard,defaults /dev/sdb "${MOUNT_DIR}"

# Grant write permissions to all users
sudo chmod a+w "${MOUNT_DIR}"
```


## Download and Unzip the AlphaFold Data 

Now you can download the databases directly to the mounted disk.

1. **Navigate to the mounted disk:**

```bash
cd "${MOUNT_DIR}"
```

### install dependicies, Git and clone the AlphaFold repository

```bash
sudo apt-get update && sudo apt-get install -y git
sudo apt install aria2
sudo apt install rsync  
sudo apt-get install tmux
```

Notes

* install `rsync` per issue https://github.com/google-deepmind/alphafold/issues/1032 

```bash
git clone https://github.com/google-deepmind/alphafold.git
```

### Full BFD dataset

1. **Run the download script:**

Naviagate to the directory, run the script and log everything

```bash
cd ./alphafold
nohup scripts/download_all_data.sh "${MOUNT_DIR}" > download.log 2>&1 &
```

To view tail of log

```bash
tail -f download.log 
 ```

And to resume a disconnected SSH session:

```bash
export PROJECT_ID="YOUIR-PROJECT-ID"
export ZONE="us-central1-a"
export INSTANCE_NAME="alphafold-data-prep-vm"
export MACHINE_TYPE="n2-standard-16" 
export DISK_SIZE_GB="5000"
export BUCKET_NAME="YOUIR-PROJECT-ID-alphafold-bucket"
export MOUNT_DIR="/mnt/disks/alphafold-data"
cd "${MOUNT_DIR}"/alphafold 
tmux 
ps aux | grep -E 'tar|aria2c|download_all_data' | grep -v grep && df -h
tail -f download.log
```

#### Small BFD dataset

Now run small download

```bash
export PROJECT_ID="YOUIR-PROJECT-ID"
export ZONE="us-central1-a"
export INSTANCE_NAME="alphafold-data-prep-vm"
export MACHINE_TYPE="n2-standard-16" 
export DISK_SIZE_GB="5000"
export BUCKET_NAME="YOUIR-PROJECT-ID-alphafold-bucket"
export MOUNT_DIR="/mnt/disks/alphafold-data"
cd "${MOUNT_DIR}"/alphafold 
tmux 
nohup scripts/download_small_bfd.sh "${MOUNT_DIR}" > download_small.log 2>&1 &
```

commands to check status on different terminal session if desired:

```bash
ps aux | grep -E 'tar|aria2c|download_small_bfd' | grep -v grep && df -h
tail -f download_small.log
```

To check status or to resume a disconnected SSH session:

```bash
export PROJECT_ID="YOUIR-PROJECT-ID"
export ZONE="us-central1-a"
export INSTANCE_NAME="alphafold-data-prep-vm"
export MACHINE_TYPE="n2-standard-16" 
export DISK_SIZE_GB="5000"
export BUCKET_NAME="YOUIR-PROJECT-ID-alphafold-bucket"
export MOUNT_DIR="/mnt/disks/alphafold-data"
cd "${MOUNT_DIR}"/alphafold 
tmux
```

commands to check status on different terminal session if desired:

```bash
ps aux | grep -E 'tar|aria2c|download_small_bfd' | grep -v grep && df -h
tail -f download.log
```

### Upload Data to Google Cloud Storage

2. **Copy the data from your disk to the bucket:**

Once the downloads are all complete, copy to the bucket:

```bash
gsutil -m cp -r "${MOUNT_DIR}/" "gs://${BUCKET_NAME}/"
```

move alphafold code repo in higher-level namespace to be safe 

```bash 
gsutil -m mv gs://${BUCKET_NAME}/alphafold-data/alphafold gs://${BUCKET_NAME}/
```


## Cleanup Resources

After you've confirmed the data is successfully in your Cloud Storage bucket, **remember to delete the Compute Engine instance and the persistent disk** to avoid ongoing charges.

1. **Delete the Compute Engine instance:**

```bash
gcloud compute instances delete "${INSTANCE_NAME}" --zone="${ZONE}" --project="${PROJECT_ID}"
```

1. **Delete the persistent disk:**

```bash
gcloud compute disks delete alphafold-data-prep-vm-1 --zone="${ZONE}" --project="${PROJECT_ID}"
```

## Terraform Quick start

* ensure user running has project owner permissions
* update /migrate terraform to workbench notebook instances since user-managed notebooks deprecated
* grant permissisions below

```bash
export PROJECT_ID=your-project-id
export PROJECT_NUMBER=your-project-number
export PROJECT_ID=$(gcloud config get-value project)
PROJECT_NUMBER=$(gcloud projects describe "${PROJECT_ID}" --format="value(projectNumber)")

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com" \
    --role="roles/artifactregistry.writer"

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-aiplatform.iam.gserviceaccount.com" \
    --role="roles/storage.admin"
```