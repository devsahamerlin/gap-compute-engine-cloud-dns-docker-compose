# How to host your applications on compute engine and configure domain name with Cloud DNS ?

Youtube Video: 

Video sur le Backend NodeJs Mongo DB: https://www.youtube.com/watch?v=hEdkuuJ_wIw&t=225s

Video sur le Frontend Angular: https://www.youtube.com/watch?v=EU40T0uDUMw



## Create a cloud storage bucket to store the startup script



```shell
gcloud auth list
gcloud config list project

export ZONE=us-east4-c
export REGION=us-east4

gcloud config set compute/zone $ZONE
gcloud config set compute/region $REGION

gcloud services enable compute.googleapis.com

export APP_NAME=emp-mng
export BUCKET_NAME=$APP_NAME-$DEVSHELL_PROJECT_ID

gsutil mb gs://$BUCKET_NAME

```

## create compose file and startup script

```shell
cat >docker-compose.yml << EOF
version: '3.7'

services:
  employees-managements-db:
    image: mongo:7.0.0
    restart: always
    ports:
      - '27017:27017'
    volumes:
      - employees-managements-data:/data/db

  employees-managements-api:
    image: devsahamerlin/employees-managements-api:latest
    restart: always
    environment:
      MONGO_CONNECTION_URL: mongodb://employees-managements-db:27017/employees-managements
    ports:
      - '5002:5002'
    depends_on:
      - employees-managements-db

  employees-managements-web:
    image: devsahamerlin/employees-managements-web:latest
    restart: always
    ports:
      - '80:80'
    depends_on:
      - employees-managements-api

volumes:
  employees-managements-data:
EOF

cat >docker-compose-playbook.yml << EOF
---
- hosts: localhost
  gather_facts: false
  check_mode: false
  become: yes
  become_method: sudo
  vars:
    container_count: 4
    default_container_name: docker
    default_container_image: ubuntu
    default_container_command: sleep 1d

  tasks:
    - name: Install aptitude
      apt:
        name: aptitude
        state: latest
        update_cache: true

    - name: Install required system packages
      apt:
        pkg:
          - apt-transport-https
          - ca-certificates
          - curl
          - software-properties-common
          - python3-pip
          - virtualenv
          - python3-setuptools
        state: latest
        update_cache: true

    - name: Add Docker GPG apt Key
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present

    - name: Add Docker Repository
      apt_repository:
        repo: deb https://download.docker.com/linux/ubuntu focal stable
        state: present

    - name: Update apt and install docker-ce
      apt:
        name: docker-ce
        state: latest
        update_cache: true

    - name: Install Docker Module for Python
      pip:
        name: docker

    - name: Install docker-compose
      get_url:
        url : https://github.com/docker/compose/releases/download/1.25.1-rc1/docker-compose-Linux-x86_64
        dest: /usr/local/bin/docker-compose
        mode: 'u+x,g+x'

    - name: Pull default Docker image
      community.docker.docker_image:
        name: "{{ default_container_image }}"
        source: pull

    - name: Create default containers
      community.docker.docker_container:
        name: "{{ default_container_name }}{{ item }}"
        image: "{{ default_container_image }}"
        command: "{{ default_container_command }}"
        state: present
      with_sequence: count={{ container_count }}
EOF

# Create a startup script to configure instances.
cat >startup-script.sh << EOF

# Install dependencies from apt
sudo apt-get update
sudo apt-get -y install traceroute iperf

sudo apt-get install ansible -y
sudo apt-get install git -y
sudo apt-get install jq -y
sudo mkdir /etc/ansible/

sudo gsutil cp gs://$BUCKET_NAME/docker-compose-playbook.yml /etc/ansible/
sudo gsutil cp gs://$BUCKET_NAME/docker-compose.yml /etc/ansible/

cd /etc/ansible/
sudo ansible-playbook docker-compose-playbook.yml
sudo docker-compose -f docker-compose.yml up -d

EOF


gsutil cp startup-script.sh gs://$BUCKET_NAME
gsutil cp docker-compose.yml gs://$BUCKET_NAME
gsutil cp docker-compose-playbook.yml gs://$BUCKET_NAME


```

## Configure the network

```shell
### create VPC
gcloud compute networks create $APP_NAME-pvc \
    --subnet-mode custom

### create subnet
gcloud compute networks subnets create $APP_NAME-$REGION-subnet \
    --network $APP_NAME-pvc \
    --range 10.0.0.0/24 \
    --region $REGION

### create firewall
gcloud compute firewall-rules create $APP_NAME-web-fw \
--allow tcp:80,tcp:5002 --network $APP_NAME-pvc --source-ranges 0.0.0.0/0 \
--target-tags backend
```

## Create instance

```shell
gcloud compute instances create $APP_NAME-vm \
--zone=$ZONE \
--machine-type=e2-medium \
--subnet $APP_NAME-$REGION-subnet \
--provisioning-model=SPOT \
--instance-termination-action=DELETE \
--tags=backend,http-server \
--metadata=startup-script-url=https://storage.googleapis.com/$BUCKET_NAME/startup-script.sh

sleep 180

```

## Configure Cloud DNS for your Domain

- We are going to use Cloud console here

## Applications source code

- Backend: https://github.com/devsahamerlin/employees-managements-api
- Frontend: https://github.com/devsahamerlin/employees-managements-app

```sh
docker build -t <docker_username>/employees-managements-api:latest .
docker build -t <docker_username>/employees-managements-web:latest .

docker push <docker_username>/employees-managements-api:latest
docker push <docker_username>/employees-managements-web:latest
```



