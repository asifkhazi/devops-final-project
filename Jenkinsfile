@Library('sharedlibrary')_

pipeline {
    environment {
        gitRepoURL = "${env.GIT_URL}"
        gitBranchName = "${env.BRANCH_NAME}"
        repoName = sh(script: "basename -s .git ${GIT_URL}", returnStdout: true).trim()
        dockerImage = "891377304789.dkr.ecr.ap-south-2.amazonaws.com/${repoName}"
        branchName = sh(script: 'echo $BRANCH_NAME | sed "s#/#-#"', returnStdout: true).trim()
        gitCommit = "${GIT_COMMIT[0..6]}"
        dockerTag = "${branchName}-${gitCommit}"
        snykOrg = "06661995-d0af-4e22-8ebb-c90e7e38f1d3"
        SCANNER_HOME=tool 'sonar-scanner'
    }
    
    agent {label 'docker'}
    stages {
        stage('Git Checkout') {
            steps {
                gitCheckout("$gitRepoURL", "refs/heads/$gitBranchName", 'githubCred')
            }
        }

        stage("Sonarqube Analysis ") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    dir('src') { 
                        sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName="$repoName" \
                        -Dsonar.projectKey="$repoName"
                        '''
                    }
                }
            }
        }

        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonarToken' 
                }
            } 
        }

        stage('Docker Build') {
            steps {
                    dockerImageBuild('$dockerImage', '$dockerTag')
            }
        }

        stage('Snyk Scan') {
            steps {
                snykImageScan('$dockerImage', '$dockerTag', 'snykCred', '$snykOrg')
            }
        }

        stage('Trivy Scan') {
            steps {
                sh "trivy image -f json -o results-${BUILD_NUMBER}.json ${dockerImage}:${dockerTag}"
            }
        }

        stage('Docker Push') {
            steps {
                dockerECRImagePush('$dockerImage', '$dockerTag', '$repoName', 'awsCred', 'ap-south-2')
            }
        }

        stage('Kubernetes Deploy - DEV') {
            when {
                branch 'development'
            }
            steps {
                kubernetesEKSHelmDeployEnv('$dockerImage', '$dockerTag', '$repoName', 'awsCred', 'ap-south-2', 'eks-cluster', 'dev')
            }
        }

        stage('Kubernetes Deploy - UAT') {        
            when {
                branch 'master_staging'
            }
            steps {
                kubernetesEKSHelmDeployEnv('$dockerImage', '$dockerTag', '$repoName', 'awsCred', 'ap-south-2', 'eks-cluster', 'uat')
            }
        }

        stage('Kubernetes Deploy - PROD') {
            when {
                branch 'main'
            }
            steps {
                kubernetesEKSHelmDeployEnv('$dockerImage', '$dockerTag', '$repoName', 'awsCred', 'ap-south-2', 'eks-cluster', 'prod')
            }
        }

    }
}
