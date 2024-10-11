pipeline {
    agent any
    environment {
        Docker_tag = ''  // Initialize the Docker_tag variable
    }
    
    stages {
        stage('Get Docker Tag') {
            steps {
                script {
                    // Fetch the Docker tag (latest commit hash)
                    Docker_tag = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                    currentBuild.displayName = "Final_Demo #${currentBuild.number} - ${Docker_tag}"
                }
            }
        }

        stage('Quality Gate Status Check') {
            agent {
                docker {
                    image 'maven'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                script {
                    withSonarQubeEnv('sonarserver') {
                        // Run SonarQube analysis
                        sh "mvn sonar:sonar"
                    }
                }
                timeout(time: 1, unit: 'HOURS') {
                    def qg = waitForQualityGate()
                    if (qg.status != 'OK') {
                        error "Pipeline aborted due to quality gate failure: ${qg.status}"
                    }
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    // Build the project using Maven
                    sh "mvn clean install"

                    // Copy the target directory to the current workspace
                    sh 'cp -r target .'

                    // Build the Docker image using the computed Docker_tag
                    sh "docker build . -t deekshithsn/devops-training:${Docker_tag}"

                    withCredentials([string(credentialsId: 'docker_password', variable: 'docker_password')]) {
                        // Login and push the Docker image
                        sh 'docker login -u deekshithsn -p $docker_password'
                        sh "docker push deekshithsn/devops-training:${Docker_tag}"
                    }
                }
            }
        }

        stage('Ansible Playbook') {
            steps {
                script {
                    // Trim the Docker tag and use it in the deployment YAML file
                    sh '''
                    final_tag=$(echo $Docker_tag | tr -d ' ')
                    sed -i "s/docker_tag/$final_tag/g" deployment.yaml
                    '''

                    // Run the Ansible playbook
                    ansiblePlaybook become: true, installation: 'ansible', inventory: 'hosts', playbook: 'ansible.yaml'
                }
            }
        }
    }
}
