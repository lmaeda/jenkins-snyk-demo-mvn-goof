pipeline {
    agent any

    // Pull your Snyk token from a Jenkins encrypted credential
    // (type "Secret text" 
    //  see "https://jenkins.io/doc/book/using/using-credentials/#adding-new-global-credentials" )
    // and put it in temporary environment variable for the Snyk CLI to consume.
    environment {
        DOCKER_ID    = credentials('DOCKER_ID')
        DOCKER_TOKEN = credentials('DOCKER_TOKEN')
        GITHUB_TOKEN = credentials('GITHUB_TOKEN')
        SNYK_TOKEN   = credentials('SNYK_TOKEN')
    }


    stages {

        stage('premerge build') {
            agent { 
                docker {
                    image 'snyk/snyk:maven-3-jdk-8'
                    args "--env SNYK_TOKEN=${SNYK_TOKEN} --env GITHUB_TOKEN=${GITHUB_TOKEN}"
                }
            }

            steps {
                echo 'Initialize & Cleanup Workspace'
                sh 'ls -la'
                sh 'rm -rf *'
                sh 'rm -rf .git'
                sh 'rm -rf .gitignore'
                sh 'ls -la'

                echo 'Git Clone'
                sh 'git config --global --add safe.directory /var/jenkins_home/workspace/SnykTSM_sample_JenkinsPipeline@2'
                git url: 'https://github.com/lmaeda/jenkins-snyk-demo-mvn-goof'
                sh 'ls -la'

                echo 'Test Build Requirements'
                sh 'java -version'
                sh 'mvn -v'
                sh 'snyk -v'

                echo 'Build'
                sh 'mvn install'
                sh 'mvn -e -X package'

                // Run snyk test to check for vulnerabilities and fail the build 
                // if any are found
                // Consider using --severity-threshold=<low|medium|high> for more 
                // granularity (see snyk help for more info).
                echo 'Snyk Test using Snyk CLI'
                sh 'snyk iac test --scan=planned-values --org=demo_high || true'
                sh 'snyk code test --org=demo_high || true'
                sh 'snyk test --all-projects --org=demo_high || true'

                // Capture the dependency tree for ongoing monitoring in Snyk.
                echo 'Snyk Monitor using Snyk CLI'
                sh 'snyk iac test --org=demo_high --scan=planned-values --report || true'
                sh 'snyk monitor  --org=demo_high --all-projects --detection-depth=8 --print-deps --remote-repo-url=jenkins_snyk_mvn_goof || true'
            }
        }

        // Capture the dependency tree for ongoing monitoring in Snyk.
        // This is typically done after deployment to some environment
        stage('Post merge build') {
            agent { 
                docker {
                    image 'snyk/snyk:docker'
                    args "--env SNYK_TOKEN=${SNYK_TOKEN} --env GITHUB_TOKEN=${GITHUB_TOKEN} --env DOCKER_ID=${DOCKER_ID} --env DOCKER_TOKEN=${DOCKER_TOKEN}"
                }
            }

            steps {
                echo 'Git Clone'
                git url: 'https://github.com/lmaeda/jenkins-snyk-demo-mvn-goof'
                sh 'ls -la'

                echo 'build container image'
                sh 'docker image build . --file Dockerfile --tag lucmaeda/my-snyk-demo-mvn-goof-jenkins:latest'

                echo 'Snyk Container Test using Snyk CLI'
                // Use your own Snyk Organization with --org=<your-org>
                sh 'docker run --env SNYK_TOKEN=${SNYK_TOKEN} -v /var/run/docker.sock:/var/run/docker.sock snyk/snyk:docker snyk container test --print-deps --org=demo_high --app-vulns --nested-jars=8 lucmaeda/my-snyk-demo-mvn-goof-jenkins:latest || true'
        
                // Capture the dependency tree for ongoing monitoring in Snyk.
                // This is typically done after deployment to some environment
                echo 'Snyk Container Monitor using Snyk CLI'
                sh 'docker run --env SNYK_TOKEN=${SNYK_TOKEN} -v /var/run/docker.sock:/var/run/docker.sock snyk/snyk:docker snyk container monitor --print-deps --org=demo_high --app-vulns --nested-jars=8 lucmaeda/my-snyk-demo-mvn-goof-jenkins:latest'
        
                echo 'Push container image'
                sh 'docker login -u lucmaeda -p ${DOCKER_TOKEN}'
                sh 'docker push lucmaeda/my-snyk-demo-mvn-goof-jenkins:latest'
            }
        }
        
    }
}
