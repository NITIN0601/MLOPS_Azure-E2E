# MLOPS_Azure-E2E

MLOps Workflow on Databricks with Azure DevOps
This repository demonstrates an end-to-end MLOps (Machine Learning Operations) workflow on Databricks using Azure DevOps. The goal is to deploy a model, along with its ancillary pipelines, to a specified Databricks workspace. The workflow utilizes Databricks Labs' dbx tool to deploy the pipelines as Databricks jobs.

Use Case: Churn Prediction
The use case presented in this repository focuses on a churn prediction problem. We leverage the IBM Telco Customer Churn dataset, which can be accessed [`here`](https://community.ibm.com/community/user/businessanalytics/blogs/steven-macko/2019/07/11/telco-customer-churn-1113). The dataset contains information about customers from a fictional telco company. Our objective is to build a simple classifier that can predict whether a customer is likely to churn.

Please note that this package is developed exclusively using an IDE, and thus, there are no Databricks Notebooks within the repository. All jobs are executed through a command line-based workflow using the dbx tool.

Workflow Overview
The MLOps workflow in this repository consists of several pipelines, each deployed as a Databricks job. These pipelines cover various stages of the model lifecycle, including model training and model deployment. The following pipelines are included:

*`Data Preprocessing Pipeline:`* This pipeline prepares the Telco Customer Churn dataset for model training. It handles tasks such as data cleaning, feature engineering, and data splitting into training and validation sets.

*`Model Training Pipeline:`* This pipeline trains a classifier using the preprocessed data from the previous pipeline. It applies a machine learning algorithm to build a churn prediction model and evaluates its performance using appropriate metrics.

*`Model Deployment Pipeline:`* This pipeline deploys the trained model to the specified Databricks workspace. It sets up the necessary infrastructure, such as creating an endpoint for serving predictions and configuring the required resources.

Each pipeline is implemented as a Databricks job, leveraging the capabilities provided by the dbx tool from Databricks Labs. The dbx tool allows for command line-based execution of jobs, enabling seamless integration with the Azure DevOps environment.

Getting Started
To get started with this MLOps workflow, follow the steps below:

Clone this repository to your local development environment.

-> Set up an Azure DevOps project and create a pipeline to execute the required jobs. Refer to the Azure DevOps documentation for detailed instructions on creating and configuring pipelines.

-> Configure the Databricks workspace connection in the Azure DevOps pipeline. Provide the necessary credentials and workspace details to enable communication between Azure DevOps and Databricks.

-> Customize the pipeline configuration files located in the pipelines directory. Adjust the parameters, file paths, and any other settings as per your specific Databricks workspace and requirements.

Execute the pipeline(s) in Azure DevOps and monitor the job statuses and logs to track the progress of the MLOps workflow.

Additional Resources
For more information on Databricks, Azure DevOps, and the tools used in this repository, refer to the following resources:

Databricks Documentation: [`docs`](https://docs.databricks.com/)


Azure DevOps Documentation: [`docs`](https://docs.microsoft.com/en-us/azure/devops/)


Databricks Labs' dbx Tool Documentation: [`dbx`](https://dbx.readthedocs.io/en/latest/index.html)


License
This repository is licensed under the MIT License. Feel free to modify and adapt the codebase to suit your needs.

Acknowledgments
We would like to acknowledge the contributors and developers who have made this MLOps workflow possible. Thank you for your efforts in advancing the field of machine learning and software engineering.

If you have any questions or encounter any issues while using this repository, please open an issue in the GitHub repository or contact us directly.

Happy MLOps-ing!



## Pipelines

The following pipelines currently defined within the package are:
- `demo-setup`
    - Deletes existing feature store tables, existing MLflow experiments and models registered to MLflow Model Registry, 
      in order to start afresh for a demo.  
- `feature-table-creation`
    - Creates new feature table and separate labels Delta table.
- `model-train`
    - Trains a scikit-learn Random Forest model  
- `model-deployment`
    - Compare the Staging versus Production models in the MLflow Model Registry. Transition the Staging model to 
      Production if outperforming the current Production model.
- `model-inference-batch`
    - Load a model from MLflow Model Registry, load features from Feature Store and score batch.

## Demo
The following outlines the workflow to demo the repo. 

### Set up
1. Fork [Fork_Github](https://github.com/NITIN0601/MLOPS_Azure-E2E)
1. Configure [Databricks CLI connection profile](https://docs.databricks.com/dev-tools/cli/index.html#connection-profiles)
    - The project is designed to use 3 different Databricks CLI connection profiles: dev, staging and prod. 
      These profiles are set in [e2e-mlops-azure/.dbx/project.json](https://github.com/NITIN0601/MLOPS_Azure-E2E/blob/main/.dbx/project.json).
    - Note that for demo purposes we use the same connection profile for each of the 3 environments. 
      **In practice each profile would correspond to separate dev, staging and prod Databricks workspaces.**
    - This [project.json](https://github.com/NITIN0601/MLOPS_Azure-E2E/blob/main/.dbx/project.json) file will have to be 
      adjusted accordingly to the connection profiles a user has configured on their local machine.
1. Configure Databricks the following variables for use in the Azure DevOps pipelines:
        - `DATABRICKS_STAGING_HOST`
            - URL of Databricks staging workspace
        - `DATABRICKS_STAGING_TOKEN`
            - [Databricks access token](https://docs.databricks.com/dev-tools/api/latest/authentication.html) for staging workspace
        - `DATABRICKS_PROD_HOST`
            - URL of Databricks production workspace
        - `DATABRICKS_PROD_TOKEN`
            - [Databricks access token](https://docs.databricks.com/dev-tools/api/latest/authentication.html) for production workspace

    #### ASIDE: Starting from scratch
    
    The following resources should not be present if starting from scratch: 
    - Feature table must be deleted
        - The table e2e_mlops_{env}.churn_features will be created when the feature-table-creation pipeline is triggered.
    - MLflow experiment
        - MLflow Experiments during model training and model deployment will be used in both the dev and prod environments. 
          The paths to these experiments are configured in their respective environment `.env` files. 
          For example, the workspace paths to use for the production environment MLflow experiments will be defined under [`./conf/prod/.prod.env`](https://github.com/NITIN0601/MLOPS_Azure-E2E/blob/main/conf/prod/.prod.env)    
        - For demo purposes, we delete these experiments if they exist to begin from a blank slate.
    - Model Registry
        - Delete Model in MLflow Model Registry if exists.
    
    **NOTE:** As part of the `initial-model-train-register` multitask job, the first task `demo-setup` will delete these, 
   as specified in [`demo_setup.yml`](https://github.com/NITIN0601/MLOPS_Azure-E2E/blob/main/conf/pipeline_configs/demo_setup.yml).

### Workflow

1. **Run `telco-churn-initial-model-train-register` multitask job**

    - To demonstrate a CICD workflow, we want to start from a “steady state” where there is a current model in production. 
      As such, we will manually trigger a multitask job to do the following steps:
      1. Set up the workspace for the demo by deleting existing MLflow experiments and register models, along with 
         existing Feature Store and labels tables. 
      1. Create a new Feature Store table to be used by the model training pipeline.
      1. Train an initial “baseline” model
    - There is then a final manual step to promote this newly trained model to production via the MLflow Model Registry UI.

    - Outlined below are the detailed steps to do this:

        1. Run the multitask `telco-churn-initial-model-train-register` job via an automated job cluster 
           (NOTE: multitask jobs can only be run via `dbx deploy; dbx launch` currently).
           ```
           dbx deploy --jobs=PROD-telco-churn-initial-model-train-register --environment=prod --files-only
           dbx launch --job=PROD-telco-churn-initial-model-train-register --environment=prod --as-run-submit --trace
           ```
           See the Limitations section below regarding running multitask jobs. In order to reduce cluster start up time
           you may want to consider using a [Databricks pool](https://docs.databricks.com/clusters/instance-pools/index.html), 
           and specify this pool ID in [`conf/deployment.yml`](https://github.com/NITIN0601/MLOPS_Azure-E2E/blob/main/conf/deployment.yml).
    - `telco-churn-initial-model-train-register` tasks:
        1. Demo setup task steps ([`demo-setup`](https://github.com/NITIN0601/MLOPS_Azure-E2E/blob/main/telco_churn/pipelines/demo_setup_job.py))
            1. Delete Model Registry model if exists (archive any existing models).
            1. Delete MLflow experiment if exists.
            1. Delete Feature Table if exists.
        1. Feature table creation task steps (`feature-table-creation`)
            1. Creates new churn_features feature table in the Feature Store
        1. Model train task steps (`model-train`)
            1. Train initial “baseline” classifier (RandomForestClassifier - `max_depth=4`) 
                - **NOTE:** no changes to config need to be made at this point
            1. Register the model. Model version 1 will be registered to `stage=None` upon successful model training.
            1. **Manual Step**: MLflow Model Registry UI promotion to `stage='Production'`
                - Go to MLflow Model Registry and manually promote model to `stage='Production'`.


2. **Code change / model update (Continuous Integration)**

    - Create new “dev/new_model” branch 
        - `git checkout -b  dev/new_model`
    - Make a change to the [`model_train.yml`](https://github.com/NITIN0601/MLOPS_Azure-E2E/blob/main/conf/pipeline_configs/model_train.yml) config file, updating `max_depth` under model_params from 4 to 8
        - Optional: change run name under mlflow params in [`model_train.yml`](https://github.com/NITIN0601/MLOPS_Azure-E2E/blob/main/conf/pipeline_configs/model_train.yml) config file
    - Create pull request, to merge the branch dev/new_model into main

* On pull request the following steps are triggered in Azure DevOps pipelines:
    1. Trigger unit tests 
    1. Trigger integration tests


3. **Cut release**

    - Create tag (e.g. `v0.0.1`)
        - `git tag <tag_name> -a -m “Message”`
            - Note that tags are matched to `v*`, i.e. `v1.0`, `v20.15.10`
    - Push tag
        - `git push origin <tag_name>`

    - On pushing this the following steps are executed as defined in `azure-pipelines.yml`:
        1. Trigger unit tests
        1. Deploy `PROD-telco-churn-model-train` job to the production environment
        1. Deploy `PROD-telco-churn-model-deployment` job to the production environment
        1. Deploy `PROD-telco-churn-model-inference-batch` job to the production environment
            - These jobs will now all be present in the specified workspace, and visible under the [Workflows](https://docs.databricks.com/data-engineering/jobs/index.html) tab.
    

4. **Run `PROD-telco-churn-model-train` job**
    - Manually trigger job via UI
        - In the Databricks workspace go to `Workflows` > `Jobs`, where the `PROD-telco-churn-model-train` job will be present.
        - Click into telco-churn-model-train and click ‘Run Now’. Doing so will trigger the job on the specified cluster configuration.
    - Alternatively you can trigger the job using the Databricks CLI:
      - `databricks jobs run-now –job-id JOB_ID`
       
    - Model train job steps (`PROD-telco-churn-model-train`)
        1. Train improved “new” classifier (RandomForestClassifier - `max_depth=8`)
        1. Register the model. Model version 2 will be registered to stage=None upon successful model training.
        1. **Manual Step**: MLflow Model Registry UI promotion to stage='Staging'
            - Go to Model registry and manually promote model to stage='Staging'

    **ASIDE:** At this point, there should now be two model versions registered in MLflow Model Registry:
        
    - Version 1 (Production): RandomForestClassifier (`max_depth=4`)
    - Version 2 (Staging): RandomForestClassifier (`max_depth=8`)


5. **Run `PROD-telco-churn-model-deployment` job (Continuous Deployment)**
    - Manually trigger job via UI
        - In the Databricks workspace go to `Workflows` > `Jobs`, where the `PROD-telco-churn-model-deployment` job will be present.
        - Click into telco-churn-model-deployment and click ‘Run Now’. Doing so will trigger the job on the specified cluster configuration. 
    - Alternatively you can trigger the job using the Databricks CLI:
      - `databricks jobs run-now –job-id JOB_ID`
    
    - Model deployment job steps  (`PROD-telco-churn-model-deployment`)
        1. Compare new “candidate model” in `stage='Staging'` versus current Production model in `stage='Production'`.
        1. Comparison criteria set through [`model_deployment.yml`](https://github.com/NITIN0601/MLOPS_Azure-E2E/blob/main/conf/pipeline_configs/model_deployment.yml)
            1. Compute predictions using both models against a specified reference dataset
            1. If Staging model performs better than Production model, promote Staging model to Production and archive existing Production model
            1. If Staging model performs worse than Production model, archive Staging model
            

6. **Run `PROD-telco-churn-model-inference-batch` job** 
    - Manually trigger job via UI
        - In the Databricks workspace go to `Workflows` > `Jobs`, where the `PROD-telco-churn-model-inference-batch` job will be present.
        - Click into telco-churn-model-inference-batch and click ‘Run Now’. Doing so will trigger the job on the specified cluster configuration.
    - Alternatively you can trigger the job using the Databricks CLI:
      - `databricks jobs run-now –job-id JOB_ID`

    - Batch model inference steps  (`PROD-telco-churn-model-inference-batch`)
        1. Load model from stage=Production in Model Registry
            - **NOTE:** model must have been logged to MLflow using the Feature Store API
        1. Use primary keys in specified inference input data to load features from feature store
        1. Apply loaded model to loaded features
        1. Write predictions to specified Delta path
    
---
## Development

While using this project, you need Python 3.X and `pip` or `conda` for package management.

### Installing project requirements

```bash
pip install -r unit-requirements.txt
```

### Install project package in a developer mode

```bash
pip install -e .
```

### Testing

#### Running unit tests

For unit testing, please use `pytest`:
```
pytest tests/unit --cov
```

Please check the directory `tests/unit` for more details on how to use unit tests.
In the `tests/unit/conftest.py` you'll also find useful testing primitives, such as local Spark instance with Delta support, local MLflow and DBUtils fixture.

#### Running integration tests

There are two options for running integration tests:

- On an interactive cluster via `dbx execute`
- On a job cluster via `dbx launch`

For quicker startup of the job clusters we recommend using instance pools 
([AWS](https://docs.databricks.com/clusters/instance-pools/index.html), 
[Azure](https://docs.microsoft.com/en-us/azure/databricks/clusters/instance-pools/), 
[GCP](https://docs.gcp.databricks.com/clusters/instance-pools/index.html)).

For an integration test on interactive cluster, use the following command:
```
dbx execute --cluster-name=<name of interactive cluster> --job=<name of the job to test>
```

For a test on an automated job cluster, deploy the job files and then launch:
```
dbx deploy --jobs=<name of the job to test> --files-only
dbx launch --job=<name of the job to test> --as-run-submit --trace
```

Please note that for testing we recommend using [jobless deployments](https://dbx.readthedocs.io/en/latest/run_submit.html), so you won't affect existing job definitions.

### Interactive execution and development on Databricks clusters

1. `dbx` expects that cluster for interactive execution supports `%pip` and `%conda` magic [commands](https://docs.databricks.com/libraries/notebooks-python-libraries.html).
2. Please configure your job in `conf/deployment.yml` file.
2. To execute the code interactively, provide either `--cluster-id` or `--cluster-name`.
```bash
dbx execute \
    --cluster-name="<some-cluster-name>" \
    --job=job-name
```

Multiple users also can use the same cluster for development. Libraries will be isolated per each execution context.

### Working with notebooks and Repos

To start working with your notebooks from [Repos](https://docs.databricks.com/repos/index.html), do the following steps:

1. Add your git provider token to your user settings
2. Add your repository to Repos. This could be done via UI, or via CLI command below:
```bash
databricks repos create --url <your repo URL> --provider <your-provider>
```
This command will create your personal repository under `/Repos/<username>/telco_churn`.
3. To set up the CI/CD pipeline with the notebook, create a separate `Staging` repo:
```bash
databricks repos create --url <your repo URL> --provider <your-provider> --path /Repos/Staging/telco_churn
```
