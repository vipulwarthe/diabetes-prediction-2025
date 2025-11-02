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

