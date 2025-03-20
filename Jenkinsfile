pipeline {
    agent {
        kubernetes {
            label 'spring-agent'
            yaml '''
            apiVersion: v1
            kind: Pod
            spec:
              containers:
              - name: maven
                image: maven:3.8.5-openjdk-17
                command:
                - /bin/sh
                - -c
                - "sleep infinity"
                workingDir: /home/jenkins/agent
                env:
                - name: HOME
                  value: /home/jenkins/agent
                volumeMounts:
                - name: workspace-volume
                  mountPath: /home/jenkins/agent
              - name: oc
                image: quay.io/openshift/origin-cli:4.15
                command:
                - /bin/sh
                - -c
                - "sleep infinity"
                workingDir: /home/jenkins/agent
                volumeMounts:
                - name: workspace-volume
                  mountPath: /home/jenkins/agent
              volumes:
              - name: workspace-volume
                emptyDir: {}
            '''
        }
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
                    sh 'mvn -B -DskipTests clean package'
                }
            }
        }
        stage('Build Image') {
            steps {
                container('oc') {
                    dir('target') {
                        script {
                            openshift.withCluster() {
                                openshift.withProject('test') {
                                    def bcExists = openshift.selector('buildconfig/spring-boot-app').exists()
                                    if (!bcExists) {
                                        sh 'oc new-build --name=spring-boot-app --binary --strategy=source --image=registry.access.redhat.com/ubi8/openjdk-17:latest --to=spring-boot-app:latest'
                                        sh 'oc start-build spring-boot-app --from-dir=. --follow'
                                    } else {
                                        def buildOutput = sh(script: 'oc start-build spring-boot-app --from-dir=. --wait --output=name', returnStdout: true).trim()
                                        def buildName = buildOutput ?: 'spring-boot-app-2'
                                        echo "Build started: ${buildName}"
                                        sh "oc logs -f ${buildName}"
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('Deploy') {
            steps {
                container('oc') {
                    script {
                        // Write DeploymentConfig with correct apiVersion
                        writeFile file: 'dc.yaml', text: '''
                        apiVersion: apps.openshift.io/v1
                        kind: DeploymentConfig
                        metadata:
                          name: spring-boot-app
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
                                name: spring-boot-app:latest
                          template:
                            metadata:
                              labels:
                                app: spring-boot-app
                            spec:
                              containers:
                              - name: spring-boot-app
                                image: spring-boot-app:latest
                                ports:
                                - containerPort: 8080
                                  protocol: TCP
                        '''
                        // Apply and rollout
                        sh 'oc apply -f dc.yaml -n test'
                        sh 'oc rollout latest dc/spring-boot-app -n test'
                        sh 'oc rollout status dc/spring-boot-app -n test'
                    }
                }
            }
        }
    }
}
