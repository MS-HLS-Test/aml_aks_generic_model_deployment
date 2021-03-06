MIT License

Copyright (c) 2021 HZ-MS-CSA

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.



# aml_model_aks_custom


1. conda env create -f project_env.yml

2. conda activate project_environment

3. Provide the necessary environment variables at run time

    1. TENANT_ID
    2. CLIENT_ID
    3. CLIENT_SECRET
    4. WORKSPACE_NAME
    5. SUBSCRIPTION_ID
    6. RESOURCE_GROUP

4. python main-generic.py (Start flask app)

5. Send a post query with the following requirements

    1. Body is set to json string with the following structure: {'data': RAW_DATA}
    2. headers = {'Content-Type':'application/json'}
    3. params = {'model_name': MODEL_NAME}
    


# Step by Step Instructions


## Set up Azure Devops Project


1. Go to [Azure Devops website](https://azure.microsoft.com/en-us/services/devops/?nav=min), and set up a project named **AML_AKS_custom_deployment** (Substitute any name as you see fit.)


## Set up Project


1. Go to Repos on the left side, and find **Import** under **Import a repository**

![](media/1.png)

2. Use https://github.com/HZ-MS-CSA/aml_aks_generic_model_deployment as clone URL


## Upload AML Model


1. As a demonstration, we will be using an onnx model from a Microsoft Cloud Workshop activity. 
    1. "This is a classification model for claim text that will predict `1` if the claim is an auto insurance claim or `0` if it is a home insurance claim. The model will be built using a type of Deep Neural Network (DNN) called the Long Short-Term Memory (LSTM) recurrent neural network using TensorFlow via the Keras library.". [Source here.](https://github.com/microsoft/MCW-Cognitive-services-and-deep-learning/blob/main/Hands-on%20lab/notebooks/03%20Claim%20Classification.ipynb)
    2. For step by step guidance on how to create and train this model, please see the [MCW workshop here.](https://github.com/microsoft/MCW-Cognitive-services-and-deep-learning) 
    3. For your convenience, you can find the onnx model under sample_model/claim_classifer.zip
    4. Download and unzip the file, and upload the onnx model to Azure ML workspace
    
    ![](media/2.png)


## Modify Azure Devops Repo Content


There are two files that need to be modified to accommodate the onnx model

1. **./main-generic.py**: This is essentially a scoring entry script that calls AML SDK, retrieve the model from the registry, and wrap it in a flask API. The original main-generic.py is a template, and you can add any relevant codes to execute the model in this file. Please replace the content of this file with **./sample_model/main-generic.py** (An example of how to customize this python script)
    1. **./sample_model/main-generic.py** is an adapted version of the original MCW-Cognitive services and deep learning Claim Classification Jupyter Notebook. [Please see source here.](https://github.com/microsoft/MCW-Cognitive-services-and-deep-learning/blob/main/Hands-on%20lab/notebooks/03%20Claim%20Classification.ipynb)
2. **./project_env.yml**: This specifies the dependencies required for the model to execute. Please replace the content of this file with **./sample_model/project_env.yml** (An example of how to customize this yml file)


## Set up Build Pipeline


1. Create a pipeline by using the classical editor. Select your Azure Repos Git as source. Then start with an empty job. 
2. Change the agent specification as **ubuntu-18.04** (same for release pipeline as well)
3. **Copy Files Activity**: Configure the activity based on the screenshot below
    ![](media/3.png)
4. **Docker-Build an Image**: Configure the activity based on the notes and screenshot below
    1. Change task version to be 0.*
    2. Select an Azure container registry, and authorize Azure Devops's Azure connection
    3. In the "Docker File" section, select the **Dockerfile** in Azure Devops repo
    4. Leave everything else as default
    ![](media/4.png)
    ![](media/5.png)
5. **Docker-Push an Image**: Configure the activity based on the notes and screenshot below
    1. Change task version to be 0.*
    2. Select the same ACR as Build an Image step above
    3. Leave everything else as default
    ![](media/6.png)
    ![](media/7.png)
6. **Publish Build Artifact**: Leave everything as default
    ![](media/8.png)
7. Save and queue the build pipeline.


## Set up Release Pipeline


1. Start with an empty job
2. Change Stage name to be AKS-Cluster-Release
    ![](media/9.png)
3. Add build artifact
    ![](media/10.png)
4. Set up continuous deployment trigger--the release pipeline will be automatically kicked off every time a build pipeline is modified
    ![](media/11.png)
5. **helm upgrade**: *Package and deploy helm charts* activity. 
    1. Select an appropriate AKS cluster
    2. Enter a custom namespace for this release. For this demo, the namespace is *aml-aks-onnx*
    3. Command is "upgrade"
    4. Chart type is "File path". Chart path is shown in the screenshot below
    5. Set release name as *aml-aks-onnx-1*
    6. Make sure to select **Install if not present** and **wait**
    7. Go to your Azure Container Registry, and find Login server URL. Your Image repository path is LOGIN_SERVER_URL/REPOSITORY_NAME. 
    7. In arguments, enter the following content:

    `--create-namespace --set image.repository=IMAGE_REPOSITORY_PATH --set image.tag=$(Build.BuildId) --set amlargs.azureTenantId=$(TenantId) --set amlargs.azureSubscriptionId=$(SubscriptionId) --set amlargs.azureResourceGroupName=$(ResourceGroup) --set amlargs.azureMlWorkspaceName=$(WorkspaceName) --set amlargs.azureMlServicePrincipalClientId=$(ClientId) --set amlargs.azureMlServicePrincipalPassword=$(ClientSecret)`

    ![](media/12.png)
    ![](media/13.png)
    ![](media/14.png)
    ![](media/15.png)
6. In Variables/Pipeline Variables, create and enter the following required values
    1. ClientId: Follow [How to: Use the portal to create an Azure AD application and service principal that can access resources](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal) to create a service principal that can access Azure ML workspace
    2. ClientSecret: See the instruction for ClientId
    3. ResourceGroup: Resource Group for AML workspace
    4. SubscriptionId: Can be found on AML worksapce overview page. 
    5. TenantId: Can be found in Azure Activate Directory
    6. WorkspaceName: AML workspace name
    
    ![](media/16.png)
7. Save, create, and deploy release
    ![](media/17.png)
    ![](media/18.png)


## Testing


1. Retrieve external IP for deployed service
    1. Open powershell
    2. `az account set --subscription SUBSCRIPTION_ID`
    3. `az aks get-credentials --resource-group RESOURCE_GROUP_NAME --name AKS_CLUSTER_NAME`
    4. `kubectl get deployments --all-namespaces=true`
    5. Find the `aml-aks-onnx` namespace, make sure it's ready
    6. `kubectl get deployments --namespace aml-aks-onnx`. External IP will be listed there
2. Use [test.ipynb](test.ipynb) to test it out
    1. endpoint is `http://EXTERNAL_IP:80/score`. You can optionally set it to be `http://EXTERNAL_IP:80/healthcheck` and then use the get method to do a quick health check
    2. In the post method section, make sure to enter the model name. In this demo, the model name is claim_classifier_onnx_demo. Enter any potential insurance claim text, and see the model classifies it into auto or home insurance claim in real time. 
    ![](media/19.png)
    ![](media/20.png)

```python

```
