// Shared Library References
def git = new org.slib.git()

// GitHub Credentials
def GITHUB_CREDS = '' 
def GITHUB_ORG = "myorg"

// Default Values
def RETAG_OLD_IMG = true
def RESTART_SERVICE = false
def AWS_TARGET_ACC = AWS_STAG_ACC
def AWS_ROLE = "JenkinsRole"
def ECS-ClusterName = "ServiceCluster" // Postfixed with '-${Environment}' e.g. ServiceCluster-Production

// AWS Environment Definitions
def AWS_PROD_ACC = [
    "AccountName": "ProdAcc",
    "AccountId": "123",
    "Environment": "Production",
    "DockerTag": "prod"]
def AWS_STAG_ACC = [
    "AccountName": "StagingAcc",
    "AccountId" : "321",
    "Environment": "Staging",
    "DockerTag": "staging"]

pipeline {
    agent { label 'AWS-ECR-ECS' }
    parameters {
        choice (name: 'ECS Service', description: 'Service to build', 
            choices: [
                'imageA',
                'imageB',
                'imageC'])
        choice (name: 'Environment', description: 'Target Environment', choices: ["${AWS_STAG_ACC.Environment}", "${AWS_PROD_ACC.Environment}"])
        string (name: 'Branch', description: 'Service Branch', defaultValue: 'master')
    }
    environment {
        ECS_SERVICE = "${params.'ECS Service'}"
        GITHUB_CART_REPO = "git@github.com:${GITHUB_ORG}/${params.'Cart Service'}.git"
        GIT_SOURCE_BRANCH = "${params.'Branch'}"
    }
    stages {
        stage('Preparing Pipeline') { 
            steps {
                cleanWs()
                script {
                    failedStage=env.STAGE_NAME
                    buildStartedBy = whoStartedJob() // Returns who/what started pipeline build (e.g. schedule)
                    if (params.Environment == AWS_PROD_ACC.Environment) {
                        AWS_TARGET_ACC = AWS_PROD_ACC // Set prod aws account if selected via params
                    }
                }
            }
        }
        stage('Checkout Service') {
            steps {
                script {
                    failedStage=env.STAGE_NAME
                    echo "Environment: ${AWS_TARGET_ACC.AccountName}"
                    gitRepo = git.checkOut(GITHUB_CART_REPO, GITHUB_CREDS, env.GIT_SOURCE_BRANCH)
                    sourceGitDetails = git.getCommitDetails()
                    env.GIT_SOURCE_COMMIT_HASH = sourceGitDetails.hash
                    env.GIT_SOURCE_COMMIT_TAG = sourceGitDetails.tag
                    echo "GIT Details: ${sourceGitDetails}"
                }
            }
        }
        stage('Inspect ECR images') {
            steps {
                script {
                    withAWS(role:"${AWS_ROLE}", roleAccount:"${AWS_TARGET_ACC.AccountId}", duration: 900, roleSessionName: 'jenkins-session')
                    {
                        ecr_images = sh (                    
                            script: "aws ecr describe-images --repository-name ${ECS_SERVICE} --output json",
                            returnStdout: true)                         
                    }
                    ecr_images_json = readJSON text: ecr_images
                    echo "All Images: ${ecr_images_json}"

                    // Look up image with the current 'in use' tag (e.g. prod, or staging)
                    arrCurrentImg = ecr_images_json.imageDetails.findAll { map -> map.imageTags?.contains(AWS_TARGET_ACC.DockerTag)}
                    currentImg = arrCurrentImg[0]
                    if (currentImg != null) {
                        echo "Prod Image: ${currentImg}"
                        echo "Image SHA with prod tag ${currentImg.imageDigest}"
                        echo "Pushed At: ${currentImg.imagePushedAt}"
                    } else {
                        echo "Currently no images tagged with ${AWS_TARGET_ACC.DockerTag}"
                        RETAG_OLD_IMG = false
                        RESTART_SERVICE = true
                    }
                }
            }
        }
        stage('Build Docker Image') { 
            steps {
                script {
                    // Build image, tag in subsequent step (if required)
                    docker_image = docker.build("${AWS_TARGET_ACC.AccountId}.dkr.ecr.ap-southeast-2.amazonaws.com/${ECS_SERVICE}")
                }
            }
        }
        stage('Push Docker Image') { 
            steps {
                script {
                    withAWS(role:"${AWS_ROLE}", roleAccount:"${AWS_TARGET_ACC.AccountId}", duration: 900, roleSessionName: 'jenkins-session')
                    {
                        docker_creds = ecrLogin()
                        sh "${docker_creds}"

                        docker_image.push("${AWS_TARGET_ACC.DockerTag}")
                        docker_image.push("${GIT_SOURCE_COMMIT_HASH}")

                        echo "tag: ${GIT_SOURCE_COMMIT_TAG}"
                        if (GIT_SOURCE_COMMIT_TAG != "null") { // Add version tag if exists
                            echo "Adding version tag to image"
                            docker_image.push("${GIT_SOURCE_COMMIT_TAG}")
                        }
                        
                        new_image = sh (                    
                            script: "aws ecr describe-images --repository-name ${ECS_SERVICE} --image-ids imageTag=${AWS_TARGET_ACC.DockerTag} --output json",
                            returnStdout: true)

                        new_image_json = readJSON text: new_image
                        echo "New image: ${new_image_json.imageDetails[0].imageDigest}"

                        if (new_image_json.imageDetails[0].imageDigest == currentImg?.imageDigest) {
                            echo "The SHA from the new image build matches that of the existing ${AWS_TARGET_ACC.DockerTag} image.  Skipping renaming '-old'"
                            RETAG_OLD_IMG = false
                        }

                        if (RETAG_OLD_IMG) {
                            sh '''
                                current_image_manifest=$(aws ecr batch-get-image --repository-name ''' + ECS_SERVICE + ''' --image-ids imageDigest=''' + currentImg.imageDigest + ''' --query images[].imageManifest --output text)
                                aws ecr put-image --repository-name ''' + ECS_SERVICE + ''' --image-tag "''' + AWS_TARGET_ACC.DockerTag + '''-old" --image-manifest "$current_image_manifest"
                            '''
                        }
                    }
                }
            }
        }
        stage('Force service refresh') {
            when {
                expression {
                    return RESTART_SERVICE == true
                }
            }
            steps {
                script {
                    echo "Forcing update of service ${ECS_SERVICE}"
                    withAWS(role:"${AWS_ROLE}", roleAccount:"${AWS_TARGET_ACC.AccountId}", duration: 900, roleSessionName: 'jenkins-session')
                    {
                        restart_output = sh (                    
                            script: "aws ecs update-service --cluster ${ECS-ClusterName}-${params.Environment} --service ${ECS_SERVICE} --force-new-deployment",
                            returnStdout: true)
                        echo "restart_output: ${restart_output}"
                    }
                }
            }
        }
    }
}
