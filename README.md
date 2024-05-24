# Terraform-Infrastructure-Provisioning-using-Error-free-Jenkins-Pipeline-with-docker-agent

Sure, here's a detailed project for setting up an error-free Jenkins pipeline with a Docker agent for Terraform. This project will guide you through configuring Jenkins, creating a Docker-based pipeline, and handling Terraform operations.

### Prerequisites:

1. **Jenkins Server**: Installed and running.
2. **Docker**: Installed on the Jenkins server.
3. **Git Repository**: Containing your Terraform configuration files.
4. **Terraform Docker Image**: Available on Docker Hub (e.g., `hashicorp/terraform`).
5. **Jenkins Plugins**: Docker Pipeline, Git Plugin, Credentials Binding Plugin.

### Project Steps:

#### Step 1: Configure Jenkins

**Install Necessary Plugins**:
- Go to `Manage Jenkins` > `Manage Plugins`.
- Install the following plugins:
  - Docker Pipeline
  - Git Plugin
  - Credentials Binding Plugin

**Configure Docker in Jenkins**:
- Go to `Manage Jenkins` > `Manage Nodes and Clouds` > `Configure Clouds`.
- Add a new Docker Cloud if necessary and configure the Docker daemon URL (usually `unix:///var/run/docker.sock`).

**Add Credentials**:
- Go to `Manage Jenkins` > `Manage Credentials`.
- Add credentials for your version control (e.g., Git) and any cloud provider (e.g., AWS) if necessary.

#### Step 2: Create a Jenkins Pipeline Job

- Go to Jenkins Dashboard.
- Click on `New Item`.
- Enter an item name, select `Pipeline`, and click `OK`.

#### Step 3: Define the Jenkinsfile

Create a `Jenkinsfile` in the root directory of your Git repository. This file defines the Jenkins pipeline stages and steps.

```groovy
pipeline {
    agent {
        docker {
            image 'hashicorp/terraform:latest'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    environment {
        TF_VAR_example = 'value' // Example of setting environment variables
        AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', git 'https://github.com/your-repo/your-terraform-project.git'
            }
        }

        stage('Terraform Init') {
            steps {
                sh 'terraform init'
            }
        }

        stage('Terraform Plan') {
            steps {
                sh 'terraform plan -out=plan.tfplan'
            }
        }

        stage('Terraform Apply') {
            steps {
                script {
                    def applyInput = input message: 'Approve the plan to apply changes?', ok: 'Apply', parameters: [booleanParam(defaultValue: false, description: 'Do you want to proceed?', name: 'Proceed')]
                    if (applyInput) {
                        sh 'terraform apply -auto-approve plan.tfplan'
                    }
                }
            }
        }

        stage('Cleanup') {
            steps {
                sh 'rm -f plan.tfplan'
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '**/*.tfplan', allowEmptyArchive: true
            cleanWs()
        }

        success {
            echo 'Pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed!'
        }
    }
}
```

### Explanation of the Jenkinsfile:

- **agent**: Specifies that the pipeline should run inside a Docker container using the `hashicorp/terraform:latest` image.
- **environment**: Sets environment variables for Terraform, including credentials from Jenkins.
- **stages**: Defines the stages of the pipeline:
  - **Checkout**: Clones the Git repository containing the Terraform configuration files.
  - **Terraform Init**: Runs `terraform init` to initialize the Terraform configuration.
  - **Terraform Plan**: Runs `terraform plan` to create an execution plan and saves it to `plan.tfplan`.
  - **Terraform Apply**: Waits for user approval and then runs `terraform apply` to apply the changes defined in `plan.tfplan`.
  - **Cleanup**: Removes the `plan.tfplan` file after applying the changes.
- **post**: Specifies actions to take at the end of the pipeline, such as archiving the Terraform plan file and cleaning up the workspace.

### Step 4: Configure the Pipeline in Jenkins

- In the Jenkins job configuration page, scroll down to the `Pipeline` section.
- Set the `Definition` field to `Pipeline script from SCM`.
- Set the `SCM` field to `Git`.
- Provide the repository URL and credentials.
- Set the `Script Path` field to `Jenkinsfile`.

### Step 5: Trigger the Pipeline

- Save the Jenkins job configuration.
- Go back to the Jenkins job dashboard.
- Click `Build Now` to trigger the pipeline.

### Additional Notes:

1. **Volume Mapping**: The `args '-v /var/run/docker.sock:/var/run/docker.sock'` argument allows Docker containers to communicate with the Docker daemon on the host, enabling nested Docker usage if needed.
2. **Credentials Management**: Use Jenkins credentials binding to securely manage sensitive information like AWS keys.
3. **Error Handling**: The `post` block ensures that artifacts are archived and the workspace is cleaned regardless of the pipeline's success or failure.
4. **Approval Step**: The `input` step requires manual intervention for applying the Terraform plan, providing an opportunity to review the plan before execution.

By following these detailed steps, you can set up an error-free Jenkins pipeline with a Docker agent for Terraform, ensuring a consistent and isolated environment for your infrastructure code. Adjust the pipeline stages and steps based on your specific use case and requirements.
