# Create-a-Virtual-Machine-in-Google-Cloud-Platform
# 0. Create ssh keys locally. located in ~/.ssh/*.pub if there is no SSH key yet, else do nothing


cat /dev/zero | ssh-keygen -t rsa -f ~/.ssh/id_rsa -C $USER@$HOSTNAME -b 2048


##### TAKS 2: Prepare Google Cloud instance

### 1. modify ssh keys and upload to gcp https://cloud.google.com/compute/docs/connect/add-ssh-keys#add_ssh_keys_to_instance_metadata

# modify by adding username to beginning of file as specified in gcp documentation
sed -i '1s/^/'"$USER:"'/' /home/$USER/.ssh/id_rsa.pub

# set variable to current project
export PROJECT_ID=$(gcloud config get-value project)


# add ssh key to metadata with the correct username
gcloud compute project-info add-metadata \
--metadata-from-file=ssh-keys=/home/$USER/.ssh/id_rsa.pub


### 2. create firewall rule to allow incoming ICMP and SSH Traffic for tag "cloud-computing"
gcloud compute \
	--project=$PROJECT_ID \
firewall-rules create cc-assignment1-firewall-rule \
	--description="Allows ICMP and SSH traffic" \
	--direction=INGRESS \
	--priority=1000 \
	--network=default \
	--action=ALLOW \
	--rules=tcp:22,icmp \
	--source-ranges=0.0.0.0/0 \
	--target-tags=cloud-computing 
	
	# use current object through PROJECT_ID
	# tcp:22 == SSH
	# source from every network

	# keep default flags as provided by gcp default 
	# instance creation for best compatibility 
	# even if some might be optional in specific usecases


### 3. create instance with follwing parameters: 

# 1) e2-standard-2
# 2) tag "cloud-computing"
# 3) image: "ubuntu server 18.04"
gcloud compute instances create cc-assignment1 \
	--project=$PROJECT_ID  \
	--zone=europe-west6-a \
	--machine-type=e2-standard-2 \
	--network-interface=network-tier=PREMIUM,subnet=default \
	--maintenance-policy=MIGRATE \
	--service-account=831303868298-compute@developer.gserviceaccount.com \
	--scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append \
	--tags=cloud-computing \
	--create-disk=auto-delete=yes,boot=yes,device-name=cc-assignment1,image=projects/ubuntu-os-cloud/global/images/ubuntu-1804-bionic-v20211103,mode=rw,size=100
	--no-shielded-secure-boot \
	--shielded-vtpm \
	--shielded-integrity-monitoring \
	--reservation-affinity=any

	# keep default flags as provided by gcp default 
	# instance creation for best compatibility 
	# even if some might be optional in specific usecases


### 4. Connect to vm and execute --comand on vm side
### --comand uploads bench.sh from local machine to vm instance
gcloud beta compute ssh --zone "europe-west6-a" "cc-assignment1"  --project $PROJECT_ID \
    --command="gcloud compute scp /home/$USER/cc-ws21-group-23/assignment1/benchmark/bench.sh --zone=europe-west6-a cc-assignment1:~ "

### 5. install newest version of sysbench
sudo curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash
sudo apt -y install sysbench

# Add cronjob: use "which sh"
echo "*/30 * * * * /bin/sh /home/$USER/bench.sh" | crontab
