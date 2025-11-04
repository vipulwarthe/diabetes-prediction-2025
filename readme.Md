to live demo: https://dibetes-prediction.streamlit.app/

## create one ec2 server with t2.medium instance type - SG- all traffic - Storage - 20

## connect your server with xshell and Run below commands after server is up and running.

- sudo apt update

- sudo apt install python3-venv -y

- python3 -m venv MLPRO

- source MLPRO/bin/activate

- git clone <repo url>

- cd foldername

- ls

- pip install - r requirements.txt

Run jupyter notebook in vscode and save the model.

- streamlit run application.py

access application on browser add the details and get the prediction for dibetes.

Enter the below values:

6	148	72	35	79	33 0	50    

you will see the result.

## Install docker on ubuntu server and create docker image and container using dockerfile.

- docker build -t diabetes-app .

- docker run -d -p 8501:8501 diabetes-app

access the app using public ip:8501 

## Install the Jenkins on ubuntu server

sudo vi jenkins.sh

sudo apt update
sudo apt install fontconfig openjdk-21-jre
java -version

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins

sudo chmod +x jenkins.sh
./jenkins.sh

sudo ufw allow 8080     (optional: If you have firewall enabled, allow traffic on port 8080)

-sudo cat /var/lib/jenkins/secrets/initialAdminPassword

automate the whole process of deployment, create jenkins cicd pipeline for application deployment using ECR and ECS.

some plugins need to install-

✅ Workspace Cleanup

✅ Docker

✅ Docker Pipeline

✅ OWASP Dependency Check

✅ AWS Credentials

✅ Credentials Binding

✅ Pipeline
✅ Pipeline: Groovy
✅ Pipeline: Stage View




