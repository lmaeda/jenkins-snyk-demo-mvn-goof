pipeline {
    agent none

    // Pull your Snyk token from a Jenkins encrypted credential
    // (type "Secret text" 
    //  see "https://jenkins.io/doc/book/using/using-credentials/#adding-new-global-credentials" )
    // and put it in temporary environment variable for the Snyk CLI to consume.
    environment {
        GITHUB_TOKEN = credentials('GITHUB_TOKEN')
        SNYK_TOKEN = credentials('SNYK_TOKEN')
    }

    stages {

        stage('premerge build') {
            agent { 
                docker {image 'snyk/snyk:maven-3-jdk-8'}
            }

            steps {
               echo 'Initialize & Cleanup Workspace'
               sh 'ls -la'
               sh 'rm -rf *'
               sh 'rm -rf .git'
               sh 'rm -rf .gitignore'
               sh 'ls -la'
            }

            steps {
                echo 'Git Clone'
                git url: 'https://github.com/lmaeda/java-goofs.git'
                sh 'ls -la'
            }

            steps {
                echo 'Test Build Requirements'
                sh 'java -version'
                sh 'mvn -v'
                sh 'snyk -v'
            }

            steps {
              echo 'Build'
              sh 'mvn install'
              sh 'mvn -e -X package'
            }

            // Run snyk test to check for vulnerabilities and fail the build 
            // if any are found
            // Consider using --severity-threshold=<low|medium|high> for more 
            // granularity (see snyk help for more info).
            steps {
                echo 'Snyk Test using Snyk CLI'
                sh './snyk iac test --scan=planned-values --org=demo_high'
                sh './snyk code test --org=demo_high'
                sh './snyk test --all-projects --org=demo_high'
            }

            // Capture the dependency tree for ongoing monitoring in Snyk.
            steps {
                echo 'Snyk Monitor using Snyk CLI'
                sh './snyk iac test --org=demo_high --scan=planned-values --report'
                sh './snyk monitor  --org=demo_high --all-projects --detection-depth=8 --print-deps --remote-repo-url=jenkins_snyk_mvn_goof'
            }
        }

        // Capture the dependency tree for ongoing monitoring in Snyk.
        // This is typically done after deployment to some environment
        stage('Post merge build') {

            agent {
                docker { image 'snyk/snyk:docker'}
            }

            environment {
                DOCKER_LOGIN = credentials('DOCKER_LOGIN')
                DOCKER_TOKEN = credentials('DOCKER_TOKEN')
            }

            steps {
                echo 'Git Clone'
                git url: 'https://github.com/lmaeda/java-goofs.git'
                sh 'ls -la'
            }

            steps {
                echo 'build container image'
                sh "docker image build . --file Dockerfile -- tag ${DOCKER_LOGIN}/my-snyk-demo-mvn-goof-jenkins:latest"
            }
            steps {
                echo 'Snyk Container Test using Snyk CLI'
                // Use your own Snyk Organization with --org=<your-org>
                sh "docker run --env ${SNYK_TOKEN} -v /var/run/docker.sock:/var/run/docker.sock snyk/snyk:docker snyk container test --print-deps --org=demo_high --app-vulns --nested-jars=8 ${DOCKER_LOGIN}/my-snyk-demo-mvn-goof-jenkins:latest"
            }
        
            // Capture the dependency tree for ongoing monitoring in Snyk.
            // This is typically done after deployment to some environment
            steps {
                echo 'Snyk Container Monitor using Snyk CLI'
                sh "docker run --env ${SNYK_TOKEN} -v /var/run/docker.sock:/var/run/docker.sock snyk/snyk:docker snyk container monitor --print-deps --org=demo_high --app-vulns --nested-jars=8 ${DOCKER_LOGIN}/my-snyk-demo-mvn-goof-jenkins:latest"
            }
        
            steps {
                echo 'Push container image'
                sh "docker push ${DOCKER_LOGIN}/my-snyk-demo-mvn-goof-jenkins:latest"
            }
        }
        
    }
}
