# Developers Guide to Velero (Mac OSX version)

Here's a handy guide to getting started developing with Velero.  This guide walks you through required tools and configurations that are needed from a developers standpoint.  This guide makes some assumptions on tools (docker, dockerhub, aws, Mac OSX).  You are free to swap in the tools of your choosing.

# Install Prerequisite tools and Configuration

## Tool versions used in creating this document
- Docker 19.03.12
- KIND 0.8.1
- Kubectl 1.18.2
- Mac OSX 10.15.6
- Velero 1.5

## Install and/or make sure docker desktop is running on the Mac
* Install docker desktop if needed
o https://www.docker.com/products/docker-desktop
* Open a terminal session on the mac




* % Docker --version


* How to check for updates and restart


* Go into "Preferences-> Command Line" and enable experimental features
o This enables "docker buildx" which is experimental in Docker 19.03  but required to build a velero container image later.
o https://docs.docker.com/buildx/working-with-buildx/
o Select "Apply & Restart" docker



* Where to get more information
o https://www.docker.com/products/docker-desktop

## Install and/or test the Velero client on the Mac
* In the terminal session
* % Brew install velero
o Make sure "brew" is installed first
o https://docs.brew.sh/Installation
* % velero version



* Brew installs the Velero client at
o /usr/local/Cellar/velero/1.5.0/bin/velero
o -and- creates a symlink at
o /usr/local/bin/velero
o Why is this useful?  If you build a new version of the Velero client and want to replace it later


## Setup a Github account
* https://docs.github.com/en/github/getting-started-with-github/set-up-git
o Follow the steps for installing the client and setting up username, commit email and credentials
* You will be using github from the terminal session
* Install the github client
* % brew install git
* % git version



* [note] need a command to make sure you can access your github account

## Setup a Dockerhub image registry account
* https://docs.docker.com/docker-hub/
* You will use this to push/pull container images from the registry
* In the terminal session
* % docker login username=yourhubusername email=youremail
o Using your dockerhub username/password
* You can also use Quay, Harbor or other image registry

## Setup an AWS account with an S3 bucket
* The AWS S3 bucket will be used to store the Velero backup
* Setup Free AWS account
* Under Services->Storage->S3



* Create a bucket which will store the velero backup



* In a terminal session on the mac, install the aws cmd line tool
* % brew install awscli
* % aws help
* Next, setup aws credentials
o This is used to make programmatic calls to aws from the aws cli



* Select "My Security Credentials"
* Under "Access Keys" select "Create New Access Key"



* Select "Download Key File" and save in a secure location on your Mac as <name>.csv
* Next use the access key to connect the aws cli on your mac to your aws account
o % aws configure



o Enter your access key id and secret access key from the .csv key file
o Select your aws region
o Enter "text" as the default output format
* Test the aws connection
o % aws s3 ls
o You should see the S3 bucket you created

o 

* [Note]  You can use another approved cloud provider besides AWS (i.e. Google Cloud, Microsoft Azure, etc)]
* In your terminal session, go to your home dir and check for the .aws credentials directory
o % cd ~
o % ls -al



## Install JSON helper utility "jq" on the Mac
* % brew install jq
* https://formulae.brew.sh/formula/jq
* This will be helpful for viewing Velero JSON files

## Install KIND ( a kinda Kubernetes version) on the Mac
* Feel free to use your own Kubernetes cluster.  This suggests KIND due to simplicity.
* You will create a Kubernetes cluster using KIND and back it up using Velero (later)
* https://kind.sigs.k8s.io/docs/user/quick-start/
* % brew install kind
* % kind version



* Create a cluster for testing the Velero backup
o % kind create cluster --name=velero-dev
* Check to make sure the cluster exists
o % kind get clusters



# Alright!  You've made it this far!  Let's get started with Velero!!!

## Install the Velero backup controller in the cluster
* Setup environment variables for AWS
* Run the velero install command
* [Note]  What does this really do?????



% Backup your cluster to AWS using Velero

* In a terminal session on the mac:
* % velero backup create <clusterlevel-v1-09092020>
* This command submits a backup request to the Velero backup controller that you installed in the cluster earlier
* The controller listens for requests and starts the backup when it sees a request
* Check to see if the backup completed



* You can also go into your AWS S3 bucket and see the backup




## Delete your cluster (Yes, really delete your Kind cluster!)
* % kind get clusters


* 
* % kind delete cluster --name=velero-dev
* % kind get clusters
o Velero-dev should be outta here!

## Restore your cluster from the Velero backup stored in AWS
* Velero backs up the kubernetes objects in the cluster.  You need an empty cluster in order to restore the backup(???)
* You can also delete an object in an existing cluster and use "velero restore" to restore the specific object
* Create an empty cluster
o % kind create cluster --name=velero-dev
* Install the Velero backup controller image in the cluster
o % velero install .
* Restore the cluster contents from backup
o % Velero restore create --from-backup <xxxx>
* % Kind get clusters


# Now let's GO dig into the development

## Velero plugins and code are developed in Go Lang
* Install Go Lang
* https://golang.org/doc/install

## Grab the Velero source code
* % git clone https://github.com/vmware-tanzu/velero.git

## Compile and build the Velero client code (Darwin/AMD64)
* cd ~/go/src/velero
* % make local
* This creates a velero executable in the current directory
* Replace the installed version 
o % sudo mv ./velero /usr/local/bin/velero
* When you run the velero command line, you should now be using your version of the velero client

[Note:  do we need to do this?  What is the output?]
## Compile and build the Velero binary targeting linux/amd64 within a build container on your local machine
* Cd ~/go/src/velero
* % make build

## Build the Velero container image
* % make container
* [Note]  You should not need Go Lang installed in order to create the container - Check this
* You should see something like this.



* Make sure the image has been built
* % docker image ls




## Push the new Velero image to your Dockerhub registry
* First, go into docker hub and create a repository for the image



* Create a tag using the image id from docker image ls
* % docker tag <image id> <dockerhubid><image name>:<version>
* % docker tag f67d7a9378bd bikeskinh/velero:v1
* Now, push the tagged image to docker hub
* % docker push bikeskinh/velero
* Go into docker hub and check that the tag has been pushed






## Use your new Velero image in place of the original Velero image you installed in the KIND cluster earlier

In this example, velero install adds the "--image" flag which uses the tagged image in dockerhub and uses your aws credentials to configure the backup controller

export VERSION=dev-dave-0914
export BUCKET=test-velero-migration
export REGION=us-east-2
## bring your own credentials - see credentials-velero.example for an example 
export SECRETFILE=credentials-velero   
export IMAGE=hub.docker.com/bikeskinh/velero:$VERSION

velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:latest \
  --bucket $BUCKET \
  --prefix $PREFIX \
  --backup-location-config region=$REGION \
  --snapshot-location-config region=$REGION \
  --secret-file $SECRETFILE \
  --image $IMAGE

# Hopefully, this gets you started! Enjoy Velero and thank you for your contribution!
