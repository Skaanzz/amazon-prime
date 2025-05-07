pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
        KUBE_CONFIG = credentials('kubeconfig')
        DOCKER_IMAGE = 'skan07/amazon-prime:latest'
    }
    stages {
        stage ("clean workspace") {
            steps {
                cleanWs()
            }
        }
        stage ("Git checkout") {
            steps {
                git branch: 'main', url: 'https://github.com/Skaanzz/amazon-prime.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=amazon-prime \
                    -Dsonar.projectKey=amazon-prime '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage("Install NPM Dependencies") {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage ("Trivy File Scan") {
            steps {
                sh "trivy fs . > trivy.txt"
            }
        }
        stage ("Build Docker Image") {
            steps {
                sh "docker build -t amazon-prime ."
            }
        }
        stage ("Tag & Push to DockerHub") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker') {
                        sh "docker tag amazon-prime skan07/amazon-prime:latest "
                        sh "docker push skan07/amazon-prime:latest "
                    }
                }
            }
        }
        stage('Generate Kubernetes Manifests') {
            steps {
                script {
                    // Create deployment.yaml
                    writeFile file: 'deployment.yaml', text: """
apiVersion: apps/v1
kind: Deployment
metadata:
  name: amazon-prime
spec:
  replicas: 2
  selector:
    matchLabels:
      app: amazon-prime
  template:
    metadata:
      labels:
        app: amazon-prime
    spec:
      imagePullSecrets:
      - name: regcred
      containers:
      - name: amazon-prime
        image: ${DOCKER_IMAGE}
        ports:
        - containerPort: 3000
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
"""
                    // Create service.yaml
                    writeFile file: 'service.yaml', text: """
apiVersion: v1
kind: Service
metadata:
  name: amazon-prime-service
spec:
  type: NodePort
  selector:
    app: amazon-prime
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
"""
                }
            }
        }

        stage('Deploy to Kubernetes') {
    steps {
        script {
            withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE'),
                            usernamePassword(
                            credentialsId: 'docker',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                            )            
                ]) {
                sh '''
                    mkdir -p ${WORKSPACE}/.kube
                    cp "${KUBECONFIG_FILE}" ${WORKSPACE}/.kube/config
                    chmod 600 ${WORKSPACE}/.kube/config
                    export KUBECONFIG=${WORKSPACE}/.kube/config
                    
                    # Create Docker registry secret
                    kubectl create secret docker-registry regcred \
                        --docker-server=docker.io \
                        --docker-username=${DOCKER_USER} \
                        --docker-password=${DOCKER_PASS} \
                        --docker-email=${DOCKER_USER}@users.noreply.docker.com \
                        --dry-run=client -o yaml | kubectl apply -f -

                    kubectl apply -f deployment.yaml
                    kubectl apply -f service.yaml
                    
                    # Increase timeout and add debugging
                    kubectl rollout status deployment/amazon-prime --timeout=5m || true
                    kubectl get pods -o wide
                    kubectl describe deployment amazon-prime
                '''
            }
        }
    }
}
}
            
    }
 
