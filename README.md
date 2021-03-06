
## Outline

- Part 1 - Launching a Jenkins Server Configured for ECR Management

- Part 2 - Prepare the Image Repository on ECR and Project Repository on GitHub with Webhook

- Part 3 - Creating Jenkins Pipeline for the Project with GitHub Webhook

- Part 4 - Cleaning up the Image Repository on AWS ECR

## Part 1 - Launching a Jenkins Server Configured for ECR Management

- Launch a pre-configured `Clarusway Jenkins Server` from the AMI of Clarusway (ami-08857fc3e51ff205e) running on Amazon Linux 2, allowing SSH (port 22) and HTTP (ports 80, 8080) connections using the [Clarusway Jenkins Server on Docker Cloudformation Template Configured to Work with ECR](./clarusway-jenkins-with-git-docker-ecr-cfn.yml). 

- Clarusway Jenkins Server is configured with admin user `admin` and password `Clarusway1234`.

- Open your Jenkins dashboard and navigate to `Manage Jenkins` >> `Manage Plugins` >> `Available` tab

- Search and select `GitHub Integration, Pipeline:GitHub, Docker, Docker Pipeline` plugins, then click to `Install without restart`. Note: No need to install the other `Git plugin` which is already installed can be seen under `Installed` tab.


## Part 2 - Prepare the Image Repository on ECR and Project Repository on GitHub with Webhook

### Step-1: Prepare the Image Repository on ECR

- Create a docker image repository `clarusway/to-do-app` on AWS ECR from Management Console.

- Create a public project repository `todo-app-node-project` on your own GitHub account.

### Step-2: Prepare Project Repository on instance

- Connect to the "Jenkins server" instance with ssh and get "sudo privileges"

```bash
ssh -i .ssh/Second.pem ec2-user@ec2-3-133-106-98.us-east-2.compute.amazonaws.com
sudo su
```

- Clone the `todo-app-node-project` repository on  your instance.

```bash
git clone https://github.com/awsdevopsteam/todo-app-node-project.git
```

### Step-3: Download the project and transfer to the project repository.

- Download the sample project `to-do-app-nodejs.tar` file from the GitHub Repo on your instance. 
  
```bash

wget https://github.com/awsdevopsteam/jenkins-first-project/raw/master/to-do-app-nodejs.tar
```
- Extract the `to-do-app-nodejs.tar` file 

```bash
tar -xvf to-do-app-nodejs.tar
```

- Check the files an folder. You will see `to-do-app-nodejs.tar`, `to-do-app-nodejs` and `todo-app-node-project`

- Copy the files from `to-do-app-nodejs` to `todo-app-node-project`

```bash
cp -R to-do-app-nodejs/* todo-app-node-project/

```

- Enter the `todo-app-node-project/` and check the files. Then commit and push them into the Github repo

```bash
git add .
git commit -m 'added todo app'
git push
if you need ---> git config --global user.email "github account email"
            ---> git config --global user.name "username"
```
- Go the github and check the changes

### Step-5: Create Token

- Add token to the github. So, go to your github Account Profile  on right of the top >>>> Settings>>>>Developer Settings>>>>>>>>Personal access tokens >>>>>> Generate new token

- Go to the >>>>>> todo-app-node-project/.git and open Git config file.

```bash
cd .git
vi config

Add "token" after "//" in the "url" part . And also paste "@" at the and of the token.
  "url = https://<yourtoken@>github.com/awsdevopsteam/todo-app-node-project.git

```

### Step-6: Create Webhook 

- Go to the `todo-app-node-project` repository page and click on `Settings`.

- Click on the `Webhooks` on the left hand menu, and then click on `Add webhook`.

- Copy the Jenkins URL from the AWS Management Console, paste it into `Payload URL` field, add `/github-webhook/` at the end of URL, and click on `Add webhook`.

```text
http://ec2-xxxxxxxx.compute-1.amazonaws.com:8080/github-webhook/
```

## Part 3 - Creating Jenkins Pipeline for the Project with GitHub Webhook

### Step-1: Github process

- Go to the Jenkins dashboard and click on `New Item` to create a pipeline.

- Enter `todo-app-pipeline` then select `Pipeline` and click `OK`.

- Enter `To Do App pipeline configured with Jenkinsfile and GitHub Webhook` in the description field.

- Put a checkmark on `GitHub Project` under `General` section, enter URL of the project repository.

```text
https://github.com/maximkim-create/200-todo-app-node-project-ac.git
```

- Put a checkmark on `GitHub hook trigger for GITScm polling` under `Build Triggers` section.

- Go to the `Pipeline` section, and select `Pipeline script from SCM` in the `Definition` field.

- Select `Git` in the `SCM` field.

- Enter URL of the project repository, and let others be default.

```text
https://github.com/maximkim-create/200-todo-app-node-project-ac.git
```

- Click `apply` and `save`. Note that the script `Jenkinsfile` should be placed under root folder of repo.

### Step-2: Jenkins instance Process

- Go to the Jenkins instance (todo-app-node-project/ directory) to create `Jenkinsfile`

```bash
cd todo-app-node-project/
ls
vi Jenkinsfile

Press "i" to edit 
```
- Create a `Jenkinsfile` within the `todo-app-node-project` repo with following pipeline script. 
```groovy
pipeline {
    agent { label "master" }
    stages {
        stage("Run app on Docker"){
            agent{
                docker{
                    image 'node:12-alpine'
                }
            }
            steps{
                withEnv(["HOME=${env.WORKSPACE}"]) {
                    sh 'yarn install --production'
                    sh 'npm install'
                }   
            }
        }
    }
}
```

- explain withEnv(["HOME=${env.WORKSPACE}"]) the meaning of this line

- once we see the code is running, lets build its image. to do this, we should write Dockerfile based and configure the Jenkinsfile

- Go under todo-app-node-project/ folder and then create a Docker file via `vi` editor.

```bash
cd todo-app-node-project/
ls
vi Dockerfile
i
Press "i" to edit 
```

- Paste  the following content within a Dockerfile which will be located in `todo-app-node-project` repository.

```dockerfile
FROM node:12-alpine
WORKDIR /app
COPY . .
RUN yarn install --production
CMD ["node", "/app/src/index.js"]
```
- Press "ESC" and ":wq " to save.

```bash
vi Jenkinsfile

Press "i" to edit 
```

```groovy

pipeline {
    agent { label "master" }
    environment {
        ECR_REGISTRY = "782349758377.dkr.ecr.us-east-1.amazonaws.com"
        APP_REPO_NAME= "clarusway-repo/todo-app"
    }
    stages {
        stage("Run app on Docker"){
            agent{
                docker{
                    image 'node:12-alpine'
                }
            }
            steps{
                withEnv(["HOME=${env.WORKSPACE}"]) {
                    sh 'yarn install --production'
                    sh 'npm install'
                }   
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build --force-rm -t "$ECR_REGISTRY/$APP_REPO_NAME:latest" .'
                sh 'docker image ls'
            }
        }
        stage('Push Image to ECR Repo') {
            steps {
                sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$ECR_REGISTRY"'
                sh 'docker push "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
            }
        }
    }
    post {
        always {
            echo 'Deleting all local images'
            sh 'docker image prune -af'
        }
    }
}

```

- Press "ESC" and ":wq " to save.

- Commit and push the local changes to update the remote repo on GitHub.

```bash
git add .
git commit -m 'added Jenkinsfile'
git push
```
- Explain, why did we get `Error: Cannot perform an interactive login from a non TTY deviceAdd` error and add the following line into ```environment``` section in the Jenkins file.

```text
PATH="/usr/local/bin/:${env.PATH}"
```

### Step-3: Jenkins Build Process

- Go to the Jenkins project page and click `Build Now`.The job has to be executed manually one time in order for the push trigger and the git repo to be registered.

### Step-4: Make change to trigger Jenkins

- Now, to trigger an automated build on Jenkins Server, we need to change code in the repo. For example, in the `src/static/js/app.js` file, update line 56 of `<p className="text-center">No items yet! Add one above!</p>` with following new text.

```html
vi src/static/js/app.js
 <p className="text-center">You have no todo task yet! Go ahead and add one!</p>
```

- Commit and push the change to the remote repo on GitHub.

```bash
git add .
git commit -m 'updated app.js'
git push
```
-if you want to manually run 

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 782349758377.dkr.ecr.us-east-1.amazonaws.com
```

- Then run the image

```bash
docker run --name todo -dp 80:3000 782349758377.dkr.ecr.us-east-1.amazonaws.com/clarusway-repo/todo-app:latest
```
- Delete the container 

```bash
docker container stop todo
docker container rm todo
```


### Step 5 Add Deploy stage 

- Go to the Jenkins instance (todo-app-node-project/ directory)to create `Jenkinsfile`
```bash
cd todo-app-node-project/
ls
vi Jenkinsfile

Press "i" to edit 
```

- Change a `Jenkinsfile` within the `todo-app-node-project` repo and add Deploy stage like below. 

- First press "dG" to delete all

```groovy
  pipeline {
    agent { label "master" }
    environment {
        ECR_REGISTRY = "782349758377.ecr.us-east-1.amazonaws.com"
        APP_REPO_NAME= "clarusway-repo/todo-app"
    }
    stages {
        stage('Build Docker Image') {
            steps {
                sh 'docker build --force-rm -t "$ECR_REGISTRY/$APP_REPO_NAME:latest" .'
                sh 'docker image ls'
            }
        }
        stage('Push Image to ECR Repo') {
            steps {
                sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$ECR_REGISTRY"'
                sh 'docker push "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
            }
        }
        stage('Deploy') {
            steps {
                sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$ECR_REGISTRY"'
                sh 'docker pull "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
                sh 'docker run --name todo -dp 80:3000 "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
            }
        }

    }
    post {
        always {
            echo 'Deleting all local images'
            sh 'docker image prune -af'
        }
    }
}
```

- Press "ESC" and ":wq " to save.

- Commit and push the local changes to update the remote repo on GitHub.

```bash
git add .
git commit -m 'added Jenkinsfile'
git push
```

### Step 6 Make change to trigger again 

- Change the script to make trigger.

```html
vi src/static/js/app.js
 <p className="text-center">You have no todo task yet! Go ahead and add one more!</p>
```

- Commit and push the change to the remote repo on GitHub.

```bash
git add .
git commit -m 'updated app.js'
git push
```

- Observe the new built triggered with `git push` command under `Build History` on the Jenkins project page.

- Explain the built results, and show the output from the shell.

- Go to Jenkins Server instance with SSH. Add `sh 'docker rm -f todo` command  before "sh `docker run.......`" command  to the `Deploy` Stage of the the Jenkinsfile 

```groovy
 
                sh 'docker rm -f todo'
 
```
- Show the result.

## Part 4 - Cleaning up the Image Repository on AWS ECR

- If necessary, authenticate the Docker CLI to your default `ECR registry`.

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com
```

- Delete Docker image from `clarusway-repo/todo-app` ECR repository from AWS CLI.

```bash
aws ecr batch-delete-image \
      --repository-name clarusway-repo/todo-app \
      --image-ids imageTag=latest \
      --region us-east-1
```

- Delete the ECR repository `clarusway-repo/todo-app` from AWS CLI.

```bash
aws ecr delete-repository \
      --repository-name clarusway/to-do-app \
      --force \
      --region us-east-1
```



<!-- Whole Jenkinsfile -->
<!-- pipeline {
    agent { label "master" }
    environment {
        ECR_REGISTRY = "782349758377.dkr.ecr.us-east-1.amazonaws.com"
        APP_REPO_NAME= "clarusway/to-do-app"
        PATH="/usr/local/bin/:${env.PATH}"
    }
    stages {
        stage("Run app on Docker"){
            agent{
                docker{
                    image 'node:12-alpine'
                }
            }
            steps{
                withEnv(["HOME=${env.WORKSPACE}"]) {
                    sh 'yarn install'
                    sh 'npm install'
                }   
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build --force-rm -t "$ECR_REGISTRY/$APP_REPO_NAME:latest" .'
                sh 'docker image ls'
            }
        }
        stage('Push Image to ECR Repo') {
            steps {
                sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$ECR_REGISTRY"'
                sh 'docker push "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
            }
        }
        stage('Deploy') {
            steps {
                sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$ECR_REGISTRY"'
                sh 'docker pull "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
                sh 'docker rm -f todo'
                sh 'docker run --name todo -dp 80:3000 "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
            }
        }
    }
    post {
        success {
            echo 'You did it. You are gonna be a good Devops'
        }
        always {
            echo 'Deleting all local images'
            sh 'docker image prune -af'
        }
    }
} -->
