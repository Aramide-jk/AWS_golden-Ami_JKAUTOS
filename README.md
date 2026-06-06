////////////////////////
MANUAL SETUP
///////////////////////

install Node.js

Connect to  instance and run:

///////////////////
curl -fsSL https://rpm.nodesource.com/setup_20.x | bash -
sudo yum install -y nodejs
node -v 

/////////////////////

Create Backend Directory

sudo mkdir -p /opt/backend
sudo chown ec2-user:ec2-user /opt/backend
///////////////////

copy backend codes;

sudo yum install -y git
cd /opt/backend
git clone <your-repo-url> .
npm install --production

npm install
npm run build

Create Script to Fetch Environment Variables from Parameter Store
Create fetch_env.sh:

sudo nano /opt/backend/fetch_env.sh
////////////////////////////
Paste:

#!/bin/bash
APP_DIR="/opt/backend"
ENV_FILE="$APP_DIR/.env"

mkdir -p $APP_DIR

ALLOWED_ORIGINS=$(aws ssm get-parameter --name "/prod/backend/ALLOWED_ORIGINS" --query "Parameter.Value" --output text)
FRONTEND_URL=$(aws ssm get-parameter --name "/prod/backend/FRONTEND_URL" --query "Parameter.Value" --output text)
FRONTEND_URL2=$(aws ssm get-parameter --name "/prod/backend/FRONTEND_URL2" --query "Parameter.Value" --output text)
JWT_SECRET=$(aws ssm get-parameter --name "/prod/backend/JWT_SECRET" --with-decryption --query "Parameter.Value" --output text)
MONGO_URI=$(aws ssm get-parameter --name "/prod/backend/MONGO_URI" --with-decryption --query "Parameter.Value" --output text)
NODE_ENV=$(aws ssm get-parameter --name "/prod/backend/NODE_ENV" --query "Parameter.Value" --output text)
PORT=$(aws ssm get-parameter --name "/prod/backend/PORT" --query "Parameter.Value" --output text)

cat <<EOF > $ENV_FILE
ALLOWED_ORIGINS=$ALLOWED_ORIGINS
FRONTEND_URL=$FRONTEND_URL
FRONTEND_URL2=$FRONTEND_URL2
JWT_SECRET=$JWT_SECRET
MONGO_URI=$MONGO_URI
NODE_ENV=$NODE_ENV
PORT=$PORT
EOF

chmod 600 $ENV_FILE
chown ec2-user:ec2-user $ENV_FILE

////////////////////////////////
Make the script executable:

sudo chmod +x /opt/backend/fetch_env.sh
//////////////////////////////////
Run the script once to generate .env:

sudo /opt/backend/fetch_env.sh
//////////////////////////////////

Verify:

cat /opt/backend/.env
////////////////////////
Identify Backend Entry Point

Check your dist/ folder:

ls /opt/backend/dist


Find the main .js file (e.g., index.js) — this will be the Node entry point.

Test manually:

cd /opt/backend
node dist/index.js


It should start without errors.

 Create Systemd Service for the Backend
sudo nano /etc/systemd/system/backend.service


Pasted:

[Unit]
Description=Node Backend Service
After=network.target

[Service]
Type=simple
User=ec2-user
WorkingDirectory=/opt/backend
ExecStart=/usr/bin/node /opt/backend/dist/index.js
Restart=always
EnvironmentFile=/opt/backend/.env

[Install]
WantedBy=multi-user.target


Replace index.js with your actual entry file if different.

Enable and Start Service

sudo systemctl daemon-reload
sudo systemctl enable backend
sudo systemctl start backend
sudo systemctl status backend


Expected output:

Active: active (running)

  Backend Health
Check .env variables:

cat /opt/backend/.env


Test the service endpoint (replace <PORT> with your app’s port):

curl localhost:<PORT>/health


//////////////////////////////////////////////////////////////////////////////
AUTMATIC(AWS AMI Builder)
////////////////////////////////////////////////////////////////////////////////

Create 4 Image Builder components:
///////////////////////////
base-os-and-tools
nodejs-20-install
backend-build
backend-systemd
///////////////////////////

Image Recipe

Base image: Amazon Linux 2023
Components (order matters):
////////////////////////
base-os-and-tools
nodejs-20-install
backend-build
backend-systemd
///////////////////////


Create Infrastructure Configuration

this policy must inlude in the selectedRoles;[EC2Role]
        azonEC2ContainerRegistryReadOnly
        AmazonEC2RoleforAWSCodeDeploy
        AmazonSSMManagedInstanceCore
        CloudWatchAgentServerPolicy
        EC2InstanceProfileForImageBuilder

        inlineRoles(check Roles[Customs])


//////////////////////

Create image pipeline(Run)

////////////////////////////////////////

Test connected instance;
////////////////////////////////////////
sudo systemctl status amazon-ssm-agent
sudo systemctl is-enabled amazon-ssm-agent
/////////////////////////////////////////////
if error;
sudo journalctl -u amazon-ssm-agent --no-pager | tail -20
////////////////////////////////////////////////////////

cat /opt/backend/.env
systemctl status backend
curl localhost:<PORT>/health


systemctl cat backend



