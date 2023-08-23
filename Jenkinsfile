/*
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-slave
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-slave-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jenkins-slave
  namespace: jenkins
*/

pipeline {

    parameters {
        choice(description: "Action", name: "Action", choices: ["Plan", "Apply", "Destroy"])
    }

    agent {
        kubernetes {
            yaml """
                apiVersion: "v1"
                kind: "Pod"
                spec:
                  securityContext:
                    runAsUser: 1001
                    runAsGroup: 1001
                    fsGroup: 1001
                  containers:
                  - command:
                    - "cat"
                    image: "alpine/git:latest"
                    imagePullPolicy: "IfNotPresent"
                    name: "git"
                    resources: {}
                    tty: true
                  - command:
                    - "cat"
                    image: "bitnami/kubectl:latest"
                    imagePullPolicy: "IfNotPresent"
                    name: "kubectl"
                    resources: {}
                    tty: true
                  serviceAccountName: jenkins-slave
            """
        }
    }

    stages {

        stage ("Git Repo") {

            steps {

                container ("git") {

                    script {
    
                        // Install repo
                        sh "git clone https://github.com/kubernetes/autoscaler"
    
                    }

                }

            }

        }

        stage ("Vertical Pod Autoscaler: Plan") {

            steps {

                container ("kubectl") {

                    script {

                        dir("autoscaler") {
                            
                            dir("vertical-pod-autoscaler") {

                                // Template
                                sh "./hack/vpa-process-yamls.sh print"

                            }

                        }
        
                    }

                }

            }

        }

        stage ("Vertical Pod Autoscaler: Apply") {

            when {
                
                expression {
                    return !env.ACTION.equals("Plan")
                }
                
            }

            steps {

                container ("kubectl") {

                    script {

                        dir("autoscaler") {
                            
                            dir("vertical-pod-autoscaler") {
                                
                                // Apply
                                if (env.ACTION.equals("Apply")) {
                                    
                                    // Run Install Script
                                    sh "./hack/vpa-up.sh"
                                    
                                // Destroy
                                } else if (env.ACTION.equals("Destroy")) {
        
                                    // Run Uninstall Script
                                    sh "./hack/vpa-down.sh"
        
                                }
                                
                            }

                        }

                    }

                }

            }

        }

    }
    
}
