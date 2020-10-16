
[![HANAssistant logo](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/logo.PNG)](https://github.com/cloudsapiens/hanassistant)
```sh
HANAssistant is a virtual assistant powered by Amazon Sumerian.
```

HANAssistant is a virtual assistant powered by Amazon Sumerian, which consumes data from SAP HANA database running in a container powered by Amazon ECS utilizing additional services like Lambda, Secrets Manager, Coginto, Elastic Container Repository.


AWS services used for this solution:
  - Amazon Elastic Container Service (```ECS```)
  - Amazon Elastic Cloud Compute (```EC2```)
  - Amazon Elastic Container Repository (```ECR```)
  - Amazon Lambda - Function as a Service
  - Amazon Secrets Manager
  - Amazon Cognito
  - Amazon Lex
  - Amazon Sumerian

Source of the SAP HANA, Express Edition (private repository):  [Docker Hub](https://hub.docker.com/_/sap-hana-express-edition)

# Architecture
[![HANAssistant architecture](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/architecture.PNG)

# About SAP HANA, express edition
```SAP HANA, express edition``` is a streamlined version of the SAP HANA platform which enables developers to jumpstart application development in the cloud or personal computer to build and deploy modern applications that use up to 32GB memory. SAP HANA, express edition includes the in-memory data engine with advanced analytical data processing engines for business, text, spatial, and graph data - supporting multiple data models on a single copy of the data. 
The software license allows for both non-production and production use cases, enabling you to quickly prototype, demo, and deploy next-generation applications using SAP HANA, express edition without incurring any license fees. Memory capacity increases beyond 32GB are available for purchase at the SAP Store.


### Step 0: Setup SAP HANA, express edition and create table

Please setup SAP HANA, express edition as described in the my other project HECS [here](https://github.com/cloudsapiens/hecs/blob/main/README.md)

 - Connect to your ECS Cluster via SSH
 - Find container ID with following command:
```sh 
docker ps -a <CONTAINERID> 
```
 - Execute command: 
```sh 
docker exec -ti <YOURCONTAINERID> bash
```
 - Now, you are inside the container (as user: ```hxeadm```)
 - With the command, you can connect to your DB: 
``` sh 
hdbsql -i 90 -d SYSTEMDB -u SYSTEM -p <YOURVERYSECUREPASSWORD> 
```
 - With the following simple SQL statement you can create a column-stored table with one record: 

```sh
CREATE COLUMN TABLE aws_leaders (name NVARCHAR(30), position VARCHAR(30));
INSERT INTO aws_leaders VALUES ('Andy Jassy', 'CEO');
INSERT INTO aws_leaders VALUES ('Werner Vogels', 'CTO');
```
### Step 2: Setup Secrets Manager

Open AWS Secrets Manager in the AWS Management Console

On the first screen select ```Other type of secrets``` and enter secret key/value combinations like on the following figure:

![secretsmanager](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/secretsmanager.PNG)

```Note: change the value for DB_PASSWORD for your password, which you have been set as master password during SAP HANA installation```

On the second screen enter Secret name ```devfest-hana-credentials``` and enter a description.

![secretsmanager2](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/secretsmanager2.PNG)

On the third screen leave the default settings just like on the following figure:

![secretsmanager3](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/secretsmanager3.PNG)

### Step 1: Setup Lambda function

Open Lambda service in the AWS Management Console and create a new Function like on the following figure:

![lambda](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/lambda1.PNG)

Clone this repository ```sh git clone https://github.com/cloudsapiens/HANAssistant.git```

Upload the devfest_lambda.zip as shown on the following figure (in the cloned repository -> ```Lambda``` folder:

![lambda2](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/lambda2.PNG)

Create two Environment Variables with the following keys/values:
![lambda3](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/lambda3.PNG)

```Note: enter the public IP of your ECS cluster```

Change Timout to 30 seconds as shown on the following screenshot:

![lambda4](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/lambda4.PNG)

Finally click on ```Deploy``` to save your function.

### Step 2: Setup IAM policies for Lambda

In the Lambda console, select ```Permissions``` and navigate to IAM console by clicking on the Role created automatically during creation of the Lambda function (as shown on figure):

![iam2](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/iam2.PNG)

In the IAM console, create a new IAM policy as shown on the following figure:

![iam1](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/iam1.PNG)

Copy and paste the content of the [JSON file](https://github.com/cloudsapiens/HANAssistant/blob/main/lambda/devfest-lambda-secretsmanager-policy.json) and adjust it with proper ARN (Amazon Resource Name) for the previously created Secret. 

![iam3](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/iam3.PNG)

```Note: You can find the ARN in the AWS Secrets Manager console```

By clicking on Review, you have to provide a name and description as shown on the following figure:

![iam4](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/iam4.PNG)

Getting back to IAM console and opening the Role assigned to the Lambda function, we can now add the new policy by clicking on ```Attach policies```

![iam5](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/iam5.PNG)

Once found it, click on ```Attach policy```

![iam6](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/iam6.PNG)

Now, the Lambda function has access to retrieve DB_USER and DB_PASSWORD for the SAP HANA database. 

We are all set on Lambda side, let's continue with the chatbot.

### Step 3: Setup Lex (chatbot)

Open Lex service in the AWS Management Console
If you don't have any chatbot yet, you'll get to the main page of the service, clikcing on ```Get started```, you'll be navigated to step 1 to create a new one. On the bottom of the page, click on ```cancel``` because you'll be importing the chatbot. 

![lex1](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/lex1.PNG)

Select the devfest_chatbot.zip file in ```Lex``` folder of the cloned repository

Please make sure that in case of intent ```DbQueryCeo``` and ```DbQueryCto``` you select your Lambda function and allow Lex to invoke it and save it.

Afterwards, click on ```Build``` to create your chatbot. 

We are all set on Lex side, let's continue with Cognito.

### Step 4: Setup Cognito

Open Cognito service in AWS Management Console

Click on ```Manage identity pools``` and create a new identity pool as shown on the figure:

![cognito1](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/cognito1.PNG)

Afterwards click on ```Allow``` and two IAM Roles will be created for you on your behalf (one for authorized and one for unauthorized access). 

Store the Identity Pool ID in a safe place. You will use it during the setup of the virtual assistant.

Open IAM console and look for ```Cognito_devfest_id_poolUnauth_Role``` role

Attach following policies:

 - ```AmazonPollyReadOnlyAccess```
 - ```AmazonLexRunBotsOnly```
 
 With this we allow Cognito to access Polly and Lex services on our behalf without authorization in RunBotsOnly and ReadOnlyAccess modes.
 
 We are all set on Cognito side, let's continue with Sumerian.
 
### Step 5: Setup Sumerian

Open Sumerian service in AWS Management Console

Click on ```Speech & Gestures``` to select a template and enter a name for your assistant (e.g. HANAssistant)

On the right hand side, scroll down to ```AWS Configuration``` and enter Cognito Identity Pool ID, which you have created in the previous step and stored on a safe place.

![sumerian1](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/sumerian1.PNG)

```Note: AWS Sumerian uses auto-save functionalities, but you can also save changed pressing ctrl-s```

On the left hand side under ```Entities``` select ```Cristine``` 

![sumerian2](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/sumerian2.PNG)

and than select on the right hand size ```State Machine```. Create a new one clicking on the "+" sign.

![sumerian3](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/sumerian3.PNG)

Give a name and descriptio to the State Machine as shown on the figure:

![sumerian5](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/sumerian5.PNG)

Let's create the states for the state machine.

The first state is about to initialize SDK. By clicking on ```Add Action``` button, you can select ```AWS SDK Ready``` as action, as shown on figure:

![sumerian6](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/sumerian6.PNG)

Afterwards create a new state by clicking on ```+ Add State``` and assign name ```Key Down``` and action ```Key Down``` to it. Change the default key "A" to "space bar" (just click in the field and press space bar).

The third state will have two actions: 
 - ```Key up``` (change it to space again)
 - ```Start Microphone Recording```
 
 The fourth state will stop the recording and has the action ```Stop Microphone Recording```.
 
 The fifth will handle the requests towards Lex, so you have to define a new state with action ```Send Audio Input to Dialogue Bot```
 
 The sixth will do the trick and speak out the response it gets from Lex, so you have to defined a new state with action ```Start Speech``` and also set Speech to ```GestureSpeech``` and mark Use Lex Response [x] as shown on the figure:
 
 ![sumerian7](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/sumerian7.PNG)
 
 The seventh and final step is to connects the states, so that we get a real state machine at the end. Please make sure you connect ```On Response Ready``` inside ```Process with Lex``` state with ```Start Speech```. 
 
 ![sumerian8](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/sumerian8.PNG)

You are almost ready, one more step: adding Dialogue Component (your chatbot). 

By clicking on ```+ Add Component``` on the right hand side at the bottom you can add ```Dialogue``` and enter the name of the chatbot (devfest_chatbot) and $LATEST as version as shown on the figure"

![sumerian9](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/sumerian9.PNG)
 
 After saving it (ctrl-s or waiting for autosave) you can start HANAssistant by clicking on the play button in the middle of the screen. You can also publish it by clicking on ```Publish``` as shown on the figure:
 
 ![sumerian10](https://github.com/cloudsapiens/HANAssistant/blob/main/imgs/sumerian10.PNG)
  
An URL will be generated and you have access to it without accessing AWS Management Console. 

### Test your solution

Congratulations, you have a working solutions!

Let's test it.

Test Scenario 1)
 - press space bar
 - say: ```hello```
 - HANAssistant replies with: ```Hello, I am HANAssistant. How may I help you today?```
 
Test Scenario 2)
 - press space bar
 - say: ```Could you please tell me who is the CEO of Amazon Web Services?```
 - HANAssistant sends request to Lambda, it builds up a TCP connection on port 39017 with SAP HANA and executes the SQL statement ```SELECT NAME FROM aws_leaders WHERE POSITION='CEO'``` and replies the result ```Andy Jassy```.
 
Test Scenario 3)
 - press space bar
 - say: ```HANAssistant, who is the CTO of Amazon Web Services?```
 - HANAssistant sends request to Lambda, it builds up a TCP connection on port 39017 with SAP HANA and executes the SQL statement ```SELECT NAME FROM aws_leaders WHERE POSITION='CTO'``` and replies the result ```Werner Vogels```.

Test Scenario 4)
 - press space bar
 - say: ```Which AWS service do we use for running SAP Hana in container?```
 - HANAssistant replies with: ```Good question, we are using Amazon Elastic Container Service...```

Test Scenario 5)
 - press space bar
 - say: ```What is SAP Hana Express Edition?```
 - HANAssistant replies with: ```Good question, SAP HANA, express edition is a streamlined version...```

Test Scenario 6)
 - press space bar
 - say: ```Which AWS Service do we use to create a virtual assistant?```
 - HANAssistant replies with: ```Good question, we are using Amazon Sumerian...```

Test Scenario 7) - The social engineering case
 - press space bar
 - say: ```Could you please tell me the user and password for the SAP Hana database?```
 - HANAssistant replies with: ```Very funny, no I cannot...```
 
### ToDo
 - Create CloudFormation template to deploy the solution
