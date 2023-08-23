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
        booleanParam(description: "Debug", name: "DEBUG", defaultValue: env.DEBUG ? env.DEBUG : "false")
        booleanParam(description: "Restart", name: "RESTART", defaultValue: env.RESTART ? env.RESTART : "false")
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
                    image: "alpine/helm:latest"
                    imagePullPolicy: "IfNotPresent"
                    name: "helm"
                    resources: {}
                    tty: true
                    volumeMounts:
                    - mountPath: "/.config"
                      name: "config-volume"
                      readOnly: false
                    - mountPath: "/.cache/helm/"
                      name: "cache-volume"
                      readOnly: false
                  - command:
                    - "cat"
                    image: "bitnami/kubectl:latest"
                    imagePullPolicy: "IfNotPresent"
                    name: "kubectl"
                    resources: {}
                    tty: true
                  serviceAccountName: jenkins-slave
                  volumes:
                  - emptyDir:
                      medium: ""
                    name: "config-volume"
                  - emptyDir:
                      medium: ""
                    name: "cache-volume"
            """
        }
    }

    stages {

        stage ("Helm Repo") {

            steps {

                container ("helm") {

                    script {
    
                        // Install repo
                        sh "helm repo add vertical-pod-autoscaler https://kubernetes.github.io/autoscaler"
                        sh "helm repo update"
    
                    }

                }

            }

        }

        stage ("Namespace: Apply") {

            when {

                expression {
                    return env.ACTION.equals("Apply")
                }

            }

            steps {

                container ("kubectl") {

                    script {

                        // Apply
                        sh "kubectl apply -f namespace.yaml"

                    }

                }

            }

        }

        stage ("Vertical Pod Autoscaler: Plan") {

            steps {

                container ("helm") {

                    script {

                        // Cluster agent options
                        VPA_OPTIONS = " "

                        // If DEBUG is enabled
                        if (env.DEBUG.equals("true")) {

                            // Enable debug
                            VPA_OPTIONS += "--debug "

                        }

                        // Template
                        sh "helm template vpa vertical-pod-autoscaler/vertical-pod-autoscaler -f vpa-values.yaml ${VPA_OPTIONS.trim()} --namespace vpa > vpa-base.yaml"

                    }

                }

                container("kustomize") {

                    script {

                        // Download envsubst
                        httpRequest url: "https://github.com/a8m/envsubst/releases/download/v1.2.0/envsubst-Linux-x86_64", outputFile: "envsubst"

                        // Add execution permission to envsubst
                        sh "chmod +x envsubst"
                        sh "ls -l -a"
    
                        sh '''
                        for file in ./custom-resource/*; do
                            ./envsubst < "${file}" > out.txt && mv out.txt "${file}";
                        done
                        '''

                        // Kustomize
                        sh "kustomize build > vpa-template.yaml"

                        // Print Yaml
                        sh "cat vpa-template.yaml"

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

                        // Apply
                        if (env.ACTION.equals("Apply")) {

                            // Apply
                            sh "kubectl apply -f vpa-template.yaml"

                        // Destroy
                        } else if (env.ACTION.equals("Destroy")) {

                            try {
                                
                                // Destroy
                                sh "kubectl delete -f vpa-template.yaml"

                            } catch (Exception e) {

                                // Do nothing

                            }

                        }

                        sh "rm vpa-template.yaml"

                    }

                }

            }

        }

        stage ("Restart") {

            when {
                
                expression {
                    return !env.ACTION.equals("Plan") && env.RESTART.equals("true")
                }
                
            }

            steps {

                container ("kubectl") {

                    script {

                        sh "kubectl rollout restart deployment -n vpa"
                        sh "kubectl rollout restart daemonset -n vpa"
                        sh "kubectl rollout restart statefulset -n vpa"

                    }

                }

            }

        }

    }
    
}
