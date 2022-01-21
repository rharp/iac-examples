pipeline {
    agent any

    tools {
        terraform 'terraform-1.1.1' // Refers to a global tool configuration for Terraform called 'terraform-1.1.1'
    }

    // Pull your Snyk token from a Jenkins encrypted credential
    // (type "Secret text"... see https://jenkins.io/doc/book/using/using-credentials/#adding-new-global-credentials)
    // and put it in temporary environment variable for the Snyk CLI to consume.
    environment {
        SNYK_TOKEN = credentials('SNYK_TOKEN') 
        SECRET_FILE_ID = credentials('AWS_TF_CREDS') // Used for secrets.auto.tfvars for secret variables to be used in aws authentication.
    }

    stages {
        
        stage('Test Build Requirements') {
            steps {
                sh 'terraform --version'
            }
        }

        stage('Initialize & Cleanup Workspace') {
            steps {
               echo 'Initialize & Cleanup Workspace'
               sh 'ls -la'
               sh 'rm -rf *'
               sh 'rm -rf .git'
               sh 'rm -rf .gitignore'
               sh 'ls -la'
            }
        }

        stage('Git Clone') {
            steps {
                git branch: 'main', url: 'https://github.com/rharp/snyk-iac-rules-example.git'
                sh 'ls -la'
            }
        }
        

        // Not required if just install the Snyk CLI on your Agent
        stage('Download Snyk CLI') {
            steps {
                sh '''
                    latest_version=$(curl -Is "https://github.com/snyk/snyk/releases/latest" | grep "^location" | sed s#.*tag/##g | tr -d "\r")
                    echo "Latest Snyk CLI Version: ${latest_version}"
                    snyk_cli_dl_linux="https://github.com/snyk/snyk/releases/download/${latest_version}/snyk-linux"
                    echo "Download URL: ${snyk_cli_dl_linux}"
                    curl -Lo ./snyk "${snyk_cli_dl_linux}"
                    chmod +x snyk
                    ls -la
                    ./snyk -v
                '''
            }
        }
        
        stage('Copy Secrets File') {
            steps {
              sh "cp ${SECRET_FILE_ID} ."
            }
        }

        stage('Build TF Plan') {
            steps {
              sh '''
              terraform init
              terraform plan -out=tfplan.binary
              terraform show -json tfplan.binary > tf-plan.json
              '''
            }
        }

        // Run snyk test to check for vulnerabilities with custom policies and fail the build if any are found
        // Consider using --severity-threshold=<low|medium|high> for more granularity (see snyk help for more info).
        stage('Snyk Test using Snyk CLI') {
            steps {
                sh './snyk iac test tf-plan.json --rules=bundle.tar.gz'
            }
        }
    }
}
