folder structure and files:



\####################################################################################################



function\_app/

&#x09;predict/

&#x09;	\_\_init\_\_.py

&#x09;	function.json

&#x09;host.json

&#x09;predict.py

&#x09;requirements.txt



\######################################################################################################



\_\_init\_\_.py

import json

import requests

import os

import azure.functions as func



def main(req: func.HttpRequest) -> func.HttpResponse:

&#x20;   try:

&#x20;       data = req.get\_json()



&#x20;       endpoint = os.environ\["ML\_ENDPOINT"]

&#x20;       key = os.environ\["ML\_KEY"]



&#x20;       headers = {

&#x20;           "Authorization": f"Bearer {key}",

&#x20;           "Content-Type": "application/json"

&#x20;       }



&#x20;       # Call ML endpoint

&#x20;       response = requests.post(endpoint, json=data, headers=headers)



&#x20;       # Return prediction

&#x20;       return func.HttpResponse(

&#x20;           response.text,

&#x20;           status\_code=200,

&#x20;           mimetype="application/json"

&#x20;       )



&#x20;   except Exception as e:

&#x20;       return func.HttpResponse(

&#x20;           json.dumps({"error": str(e)}),

&#x20;           status\_code=500,

&#x20;           mimetype="application/json"

&#x20;       )

&#x09;	







function.json

{

&#x20; "bindings": \[

&#x20;   {

&#x20;     "authLevel": "function",

&#x20;     "type": "httpTrigger",

&#x20;     "direction": "in",

&#x20;     "name": "req",

&#x20;     "methods": \["post"]

&#x20;   },

&#x20;   {

&#x20;     "type": "http",

&#x20;     "direction": "out",

&#x20;     "name": "$return"

&#x20;   }

&#x20; ]

}











host.json

{

&#x20; "version": "2.0"

}









requirements.txt

azure-functions

requests







\############################################################################################################

IN NOTEBOOK MAKE THE FOLLOWING FILES:

1. environment.yml
2. score.py
3. housing.csv



\###########################################################################################################

environment.yml

name: housing-env

channels:

&#x20; - defaults

dependencies:

&#x20; - python=3.8

&#x20; - pip

&#x20; - pip:

&#x20;     - numpy

&#x20;     - pandas

&#x20;     - scikit-learn

&#x20;     - joblib

&#x20;     - azureml-inference-server-http





housing.csv

area,bedrooms,bathrooms,stories,parking,price

1200,2,1,1,1,300000

1500,3,2,2,1,500000

1800,3,2,2,2,600000

2000,4,3,2,2,750000

800,1,1,1,0,200000

2200,4,3,3,2,850000

1600,3,2,2,1,550000

1400,2,2,1,1,400000

1700,3,2,2,2,620000

900,2,1,1,0,250000

2500,4,3,3,2,900000

3000,5,4,3,3,1200000

1100,2,1,1,1,320000

1300,2,2,1,1,380000

2100,4,3,2,2,780000

1900,3,2,2,2,670000

1000,2,1,1,1,280000

2300,4,3,3,2,870000

2600,5,4,3,3,950000

2800,5,4,3,3,1100000



score.py

import json

import joblib

import numpy as np

import os



def init():

&#x20;   global model

&#x20;   model\_path = os.path.join(os.getenv("AZUREML\_MODEL\_DIR"), "model.pkl")

&#x20;   model = joblib.load(model\_path)



def run(data):

&#x20;   try:

&#x20;       if isinstance(data, str):

&#x20;           data = json.loads(data)



&#x20;       features = np.array(\[\[

&#x20;           data\['area'],

&#x20;           data\['bedrooms'],

&#x20;           data\['bathrooms'],

&#x20;           data\['stories'],

&#x20;           data\['parking']

&#x20;       ]])



&#x20;       prediction = model.predict(features)



&#x20;       return {"prediction": float(prediction\[0])}



&#x20;   except Exception as e:

&#x20;       return {"error": str(e)}



\###################################################################################################################



THEN IN ANOTHER IPYNB FILE RUN THE FOLLOWING CODE CELL BY CELL



\#TRAIN

import pandas as pd

from sklearn.linear\_model import LinearRegression

import joblib



\# Load dataset

df = pd.read\_csv("housing.csv")



\# Select features (adjust if needed)

X = df\[\['area', 'bedrooms', 'bathrooms', 'stories', 'parking']]

y = df\['price']



\# Train model

model = LinearRegression()

model.fit(X, y)



\# Save model

joblib.dump(model, "model.pkl")



print("✅ Model trained and saved as model.pkl")





\#################################################

pip install azure-ai-ml azure-identity







\#################################################

from azure.ai.ml import MLClient

from azure.ai.ml.entities import (

&#x20;   ManagedOnlineEndpoint,

&#x20;   ManagedOnlineDeployment,

&#x20;   Environment,

&#x20;   Model,

&#x20;   CodeConfiguration

)

from azure.identity import DefaultAzureCredential



\# Connect to workspace

ml\_client = MLClient.from\_config(credential=DefaultAzureCredential())



\# Register model

model = ml\_client.models.create\_or\_update(

&#x20;   Model(

&#x20;       path="model.pkl",

&#x20;       name="housing-model"

&#x20;   )

)



\# Create environment

env = ml\_client.environments.create\_or\_update(

&#x20;   Environment(

&#x20;       name="housing-env",

&#x20;       conda\_file="environment.yml",

&#x20;       image="mcr.microsoft.com/azureml/minimal-ubuntu20.04-py38-cpu-inference:latest"

&#x20;   )

)



\# Create endpoint

endpoint = ManagedOnlineEndpoint(

&#x20;   name="housing-endpoint-v2",

&#x20;   auth\_mode="key"

)



ml\_client.begin\_create\_or\_update(endpoint).result()



\# Create deployment

deployment = ManagedOnlineDeployment(

&#x20;   name="blue",

&#x20;   endpoint\_name="housing-endpoint-v2",

&#x20;   model=model,

&#x20;   environment=env,

&#x20;   code\_configuration=CodeConfiguration(

&#x20;       code="./",

&#x20;       scoring\_script="score.py"

&#x20;   ),

&#x20;   instance\_type="Standard\_DS1\_v2",

&#x20;   instance\_count=1

)



ml\_client.begin\_create\_or\_update(deployment).result()



\# Route traffic

endpoint.traffic = {"blue": 100}

ml\_client.begin\_create\_or\_update(endpoint).result()



print("🚀 Endpoint deployed!")



\#####IT WILL TAKE 15 MINS##########################################





endpoint = ml\_client.online\_endpoints.get("housing-endpoint-v2")

print(endpoint.scoring\_uri)





\########COPY THE LINK AND REMEMBER#################################



endpoint = ml\_client.online\_endpoints.get("housing-endpoint-v2")



keys = ml\_client.online\_endpoints.get\_keys("housing-endpoint-v2")



print("🔑 Primary Key:", keys.primary\_key)

print("🔑 Secondary Key:", keys.secondary\_key)



\##############MAKE NOTE OF THE PRIMARY KEY#########################



NOW CREATE AN AZURE FUNCTION BY CLICKING ON CREATE NEW RESOURCE AND THEN FUNCION APP THEN MAKE SURE TO CHOOSE "APP SERVICE" AS THE HOSTING PLAN AND THEN IN THE RUNTIME CHOOSE PYTHON  ADN THE VERSION TO BE PYTHON 3.10 AND THEN CREATE.



IN THE NEWLY MADE FUNCTION APP GO TO SETTINGS GO TO ENVIRONMENT VARIABLES AND THEN ADD THE FOLLWING KEYS AND VALUES

1. ML\_ENDPOINT : "THE ENPOINT LINK REMEBERED ABOVE"
2. ML\_KEY : "THE PRIMARY KEY TO BE REMEBERED ABOVE"



CLICK SAVE.



THEN ZIP ALL THE FILES IN THE FUNCTION\_APP FOLDER ABOVE.



THEN GO TO DEPLOYMENT AND THEN TO DEPLOYMENT CENTER AND THEN ON THE SOURCE DROP DOWN CHOOSE PUBLISH FILES(NEW) HERE CHOOSE THE ZIP FILE AND UPLOAD AND WAIT FOR THE NOTFICIATION OF CODE SAVED SUCCESSFULLY.



NOW GO TO OVERVIEW THEN CLICK THE FUNCTION PREDICT THEN CLICK THE "GET FUNCTION URL" AND COPY THE FIRST LINK.



NOW GO TO POSTMAN OR REQBIN AND THEN PASTE THE COPIED LINK AND THEN CHOOSE POST METHOD AND THEN CHOOSE BODY AND THEN PASTE THE FOLLOWING UNDER THE JSON SUBHEADING

{

&#x20; "area": 1200,

&#x20; "bedrooms": 3,

&#x20; "bathrooms": 2,

&#x20; "stories": 2,

&#x20; "parking": 1

}



THEN CLICK SEND U WILL RECIEVE RESPONSE AS

{

&#x20;   "prediction": 405826.596682115

}





&#x20;

