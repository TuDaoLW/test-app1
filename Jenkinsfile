pipeline {
    agent {
        kubernetes {
            label 'spring-agent'
            // Uses jenkins-ns:jenkins
            //namespace: 'kube-system'        // Run pod in jenkins-ns
            yaml '''
            apiVersion: v1
            kind: Pod
            spec:
              serviceAccountName: 'jenkins'
              containers:
              - name: maven
                image: maven:3.8.5-openjdk-17
                command: ["/bin/sh", "-c", "sleep infinity"]
                workingDir: /home/jenkins/agent
                env:
                - name: HOME
                  value: /home/jenkins/agent
                volumeMounts:
                - name: workspace-volume
                  mountPath: /home/jenkins/agent
              - name: oc
                image: quay.io/openshift/origin-cli:4.15
                command: ["/bin/sh", "-c", "sleep infinity"]
                workingDir: /home/jenkins/agent
                volumeMounts:
                - name: workspace-volume
                  mountPath: /home/jenkins/agent
              - name: sonar
                image: sonarsource/sonar-scanner-cli:5.0
                command: ["/bin/sh", "-c", "sleep infinity"]
                volumeMounts:
                - name: workspace-volume
                  mountPath: /home/jenkins/agent
              - name: db
                image: postgres:13
                env:
                - name: POSTGRES_USER
                  value: "testuser"
                - name: POSTGRES_PASSWORD
                  value: "testpass"
                - name: POSTGRES_DB
                  value: "testdb"
                command: ["/bin/sh", "-c", "sleep infinity"]
                volumeMounts:
                - name: workspace-volume
                  mountPath: /home/jenkins/agent
              volumes:
              - name: workspace-volume
                emptyDir: {}
            '''
        }
    }
    triggers { pollSCM('H/5 * * * *') }  // Poll every 5 minutes
    environment {
        STAGING_NS = 'test'
        IMAGE_NAME = "spring-boot-app:${env.BUILD_NUMBER}"
        SONAR_HOST = 'http://sonarqube-sonarqube.kube-system.svc:9000'  // Adjust to your SonarQube URL
        SONAR_TOKEN = 'squ_dc4b4d10d93c2dfd06a3df225a865497663d2bcb'  // Store token in Jenkins credentials
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn -B clean package -DskipTests'
                }
            }
        }
        stage('Unit Tests') {
            steps {
                container('maven') {
                    sh 'mvn test'
                }
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        stage('Security Check') {
            steps {
                container('sonar') {
                    sh """
                    sonar-scanner \
                      -Dsonar.projectKey=demo-app \
                      -Dsonar.sources=. \
                      -Dsonar.exclusions=target/** \
                      -Dsonar.java.binaries=target/classes \
                      -Dsonar.host.url=${SONAR_HOST} \
                      -Dsonar.token=${SONAR_TOKEN}
                    """
                }
            }
        }
        
        stage('Build Image') {
            steps {
                container('oc') {
                    dir('target') {
                        script {
                            def bcExists = sh(script: "oc get bc/spring-boot-app -n ${STAGING_NS} --no-headers | wc -l", returnStdout: true).trim() != '0'
                            if (!bcExists) {
                                sh "oc new-build --name=spring-boot-app --binary --strategy=source --image=registry.access.redhat.com/ubi8/openjdk-17:latest --to=${IMAGE_NAME} -n ${STAGING_NS}"
                            }
                            def buildOutput = sh(script: "oc start-build spring-boot-app --from-dir=. --wait --output=name -n ${STAGING_NS}", returnStdout: true).trim()
                            def buildName = buildOutput ?: "spring-boot-app-${env.BUILD_NUMBER}"
                            sh "oc logs -f ${buildName} -n ${STAGING_NS}"
                        }
                    }
                }
            }
        }
        stage('Deploy to Staging') {
            steps {
                container('oc') {
                    script {
                        writeFile file: 'dc-staging.yaml', text: """
                        apiVersion: apps.openshift.io/v1
                        kind: DeploymentConfig
                        metadata:
                          name: spring-boot-app
                          namespace: ${STAGING_NS}
                        spec:
                          replicas: 1
                          selector:
                            app: spring-boot-app
                          triggers:
                          - type: ImageChange
                            imageChangeParams:
                              automatic: true
                              containerNames:
                              - spring-boot-app
                              from:
                                kind: ImageStreamTag
                                name: ${IMAGE_NAME}
                          template:
                            metadata:
                              labels:
                                app: spring-boot-app
                            spec:
                              containers:
                              - name: spring-boot-app
                                image: ${IMAGE_NAME}
                                ports:
                                - containerPort: 8080
                        """
                        sh "oc apply -f dc-staging.yaml -n ${STAGING_NS}"
                        sh "oc rollout status dc/spring-boot-app -n ${STAGING_NS}"
                        sh "oc expose svc/spring-boot-app --name=staging-route -n ${STAGING_NS} || true"
                    }
                }
            }
        }
    }
    post {
        always {
            archiveArtifacts 'target/*.jar'
        }
    }
}
