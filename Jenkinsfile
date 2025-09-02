pipeline {
    agent any
    
    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'stage', 'prod'],
            description: 'Target environment for deployment'
        )
        choice(
            name: 'ACTION',
            choices: ['deploy', 'request-infra', 'deploy-with-infra'],
            description: 'Action to perform'
        )
        string(
            name: 'IMAGE_TAG',
            defaultValue: 'latest',
            description: 'Docker image tag to deploy'
        )
        booleanParam(
            name: 'DRY_RUN',
            defaultValue: false,
            description: 'Perform a dry run without making changes'
        )
    }
    
    environment {
        AWS_DEFAULT_REGION = 'us-east-2'
        HELM_NAMESPACE = "myapp"
        CHART_PATH = "./charts/myapp"
        INFRASTRUCTURE_REPO = "infrastructure-platform-devops"
        INFRASTRUCTURE_BRANCH = "main"
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
                    // Clean workspace
                    cleanWs()
                    
                    // Checkout application code
                    checkout scm
                    
                    // Set build metadata
                    env.BUILD_TIMESTAMP = sh(
                        script: "date +%Y%m%d-%H%M%S",
                        returnStdout: true
                    ).trim()
                    
                    env.GIT_SHORT_COMMIT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                    
                    // Set image tag if not provided
                    if (params.IMAGE_TAG == 'latest') {
                        env.FINAL_IMAGE_TAG = "${env.GIT_SHORT_COMMIT}-${env.BUILD_TIMESTAMP}"
                    } else {
                        env.FINAL_IMAGE_TAG = params.IMAGE_TAG
                    }
                }
            }
        }
        
        stage('Validate Helm Chart') {
            steps {
                script {
                    echo "🔍 Validating Helm chart..."
                    
                    // Lint the Helm chart
                    sh """
                        helm lint ${env.CHART_PATH} \
                            --values ${env.CHART_PATH}/values.yaml \
                            --values env/${params.ENVIRONMENT}/values.yaml
                    """
                    
                    // Template and validate
                    sh """
                        helm template myapp ${env.CHART_PATH} \
                            --values ${env.CHART_PATH}/values.yaml \
                            --values env/${params.ENVIRONMENT}/values.yaml \
                            --set image.tag=${env.FINAL_IMAGE_TAG} \
                            --namespace ${env.HELM_NAMESPACE} \
                            --dry-run > /tmp/helm-output.yaml
                    """
                    
                    // Validate Kubernetes manifests
                    sh """
                        kubectl --dry-run=client apply -f /tmp/helm-output.yaml
                    """
                }
            }
        }
        
        stage('Request Infrastructure') {
            when {
                anyOf {
                    expression { params.ACTION == 'request-infra' }
                    expression { params.ACTION == 'deploy-with-infra' }
                }
            }
            steps {
                script {
                    echo "🏗️ Requesting infrastructure for ${params.ENVIRONMENT}..."
                    
                    // Check if infrastructure request file exists
                    if (fileExists("infra/requests/${params.ENVIRONMENT}.yaml")) {
                        echo "📋 Infrastructure request file found"
                        
                        // Display the infrastructure request
                        sh "cat infra/requests/${params.ENVIRONMENT}.yaml"
                        
                        // Trigger infrastructure runner job
                        def infrastructureJob = build(
                            job: "${env.INFRASTRUCTURE_REPO}/runner",
                            parameters: [
                                string(name: 'APP_NAME', value: 'myapp'),
                                string(name: 'ENVIRONMENT', value: params.ENVIRONMENT),
                                string(name: 'REQUEST_FILE_PATH', value: "infra/requests/${params.ENVIRONMENT}.yaml"),
                                string(name: 'SOURCE_REPO', value: env.JOB_NAME),
                                string(name: 'SOURCE_COMMIT', value: env.GIT_SHORT_COMMIT),
                                booleanParam(name: 'DRY_RUN', value: params.DRY_RUN)
                            ],
                            wait: true,
                            propagate: true
                        )
                        
                        // Get infrastructure outputs
                        env.INFRA_JOB_NUMBER = infrastructureJob.number
                        echo "✅ Infrastructure job completed: #${env.INFRA_JOB_NUMBER}"
                        
                        // Copy artifacts from infrastructure job
                        copyArtifacts(
                            projectName: "${env.INFRASTRUCTURE_REPO}/runner",
                            selector: specific("${env.INFRA_JOB_NUMBER}"),
                            filter: "outputs/${params.ENVIRONMENT}/myapp/**",
                            target: "infra-outputs"
                        )
                        
                        // Extract IRSA role ARN from outputs
                        if (fileExists("infra-outputs/outputs/${params.ENVIRONMENT}/myapp/irsa-role-arn.txt")) {
                            env.IRSA_ROLE_ARN = readFile("infra-outputs/outputs/${params.ENVIRONMENT}/myapp/irsa-role-arn.txt").trim()
                            echo "🔑 IRSA Role ARN: ${env.IRSA_ROLE_ARN}"
                        }
                        
                    } else {
                        echo "⚠️ No infrastructure request file found at infra/requests/${params.ENVIRONMENT}.yaml"
                        echo "ℹ️ Proceeding without infrastructure provisioning"
                    }
                }
            }
        }
        
        stage('Deploy Application') {
            when {
                anyOf {
                    expression { params.ACTION == 'deploy' }
                    expression { params.ACTION == 'deploy-with-infra' }
                }
            }
            steps {
                script {
                    echo "🚀 Deploying application to ${params.ENVIRONMENT}..."
                    
                    // Prepare deployment command
                    def helmCommand = """
                        helm upgrade --install myapp ${env.CHART_PATH} \\
                            --namespace ${env.HELM_NAMESPACE} \\
                            --create-namespace \\
                            --values ${env.CHART_PATH}/values.yaml \\
                            --values env/${params.ENVIRONMENT}/values.yaml \\
                            --set image.tag=${env.FINAL_IMAGE_TAG} \\
                            --set-string labels.version=${env.FINAL_IMAGE_TAG} \\
                            --set-string labels.commit=${env.GIT_SHORT_COMMIT} \\
                            --timeout 10m \\
                            --wait
                    """
                    
                    // Add IRSA role ARN if available
                    if (env.IRSA_ROLE_ARN) {
                        helmCommand += " --set irsa.roleArn=${env.IRSA_ROLE_ARN}"
                    }
                    
                    // Add dry-run flag if needed
                    if (params.DRY_RUN) {
                        helmCommand += " --dry-run"
                    }
                    
                    // Execute deployment
                    sh helmCommand
                    
                    if (!params.DRY_RUN) {
                        echo "✅ Deployment completed successfully"
                        
                        // Wait for rollout to complete
                        sh """
                            kubectl rollout status deployment/myapp \\
                                --namespace ${env.HELM_NAMESPACE} \\
                                --timeout=600s
                        """
                        
                        // Get deployment info
                        sh """
                            kubectl get pods,svc,ingress \\
                                --namespace ${env.HELM_NAMESPACE} \\
                                -l app.kubernetes.io/name=myapp
                        """
                    }
                }
            }
        }
        
        stage('Smoke Tests') {
            when {
                allOf {
                    not { params.DRY_RUN }
                    anyOf {
                        expression { params.ACTION == 'deploy' }
                        expression { params.ACTION == 'deploy-with-infra' }
                    }
                }
            }
            steps {
                script {
                    echo "🧪 Running smoke tests..."
                    
                    // Get the ingress URL
                    def ingressHost = sh(
                        script: """
                            kubectl get ingress myapp \\
                                --namespace ${env.HELM_NAMESPACE} \\
                                -o jsonpath='{.spec.rules[0].host}' 2>/dev/null || echo ""
                        """,
                        returnStdout: true
                    ).trim()
                    
                    if (ingressHost) {
                        echo "🌐 Testing application at: https://${ingressHost}"
                        
                        // Basic health check
                        sh """
                            curl -f -s -S https://${ingressHost}/healthz || {
                                echo "❌ Health check failed"
                                exit 1
                            }
                        """
                        
                        // Ready check
                        sh """
                            curl -f -s -S https://${ingressHost}/readyz || {
                                echo "❌ Ready check failed"
                                exit 1
                            }
                        """
                        
                        echo "✅ Smoke tests passed"
                    } else {
                        echo "⚠️ No ingress found, skipping external tests"
                        
                        // Internal service test via port-forward
                        sh """
                            kubectl port-forward service/myapp 8080:80 \\
                                --namespace ${env.HELM_NAMESPACE} &
                            PF_PID=\$!
                            sleep 5
                            
                            curl -f -s -S http://localhost:8080/healthz || {
                                kill \$PF_PID
                                echo "❌ Internal health check failed"
                                exit 1
                            }
                            
                            kill \$PF_PID
                            echo "✅ Internal smoke tests passed"
                        """
                    }
                }
            }
        }
        
        stage('Production Approval') {
            when {
                allOf {
                    expression { params.ENVIRONMENT == 'prod' }
                    not { params.DRY_RUN }
                }
            }
            steps {
                script {
                    echo "⏸️ Production deployment requires approval"
                    
                    def deploymentSummary = """
                    🚀 Production Deployment Summary:
                    
                    Application: myapp
                    Environment: ${params.ENVIRONMENT}
                    Image Tag: ${env.FINAL_IMAGE_TAG}
                    Git Commit: ${env.GIT_SHORT_COMMIT}
                    Timestamp: ${env.BUILD_TIMESTAMP}
                    
                    Infrastructure: ${env.IRSA_ROLE_ARN ? 'Provisioned' : 'Not Required'}
                    ${env.IRSA_ROLE_ARN ? 'IRSA Role: ' + env.IRSA_ROLE_ARN : ''}
                    """
                    
                    input(
                        message: deploymentSummary,
                        ok: 'Approve Production Deployment',
                        submitterParameter: 'APPROVER'
                    )
                    
                    echo "✅ Production deployment approved by: ${env.APPROVER}"
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo "🧹 Cleaning up..."
                
                // Archive important files
                archiveArtifacts(
                    artifacts: 'env/**/*.yaml,infra/requests/**/*.yaml,infra-outputs/**/*',
                    allowEmptyArchive: true,
                    fingerprint: true
                )
                
                // Clean up temporary files
                sh "rm -f /tmp/helm-output.yaml"
            }
        }
        
        success {
            script {
                def message = """
                ✅ **Deployment Successful**
                
                **Application:** myapp
                **Environment:** ${params.ENVIRONMENT}
                **Action:** ${params.ACTION}
                **Image Tag:** ${env.FINAL_IMAGE_TAG}
                **Git Commit:** ${env.GIT_SHORT_COMMIT}
                **Build:** #${env.BUILD_NUMBER}
                ${params.DRY_RUN ? '**Mode:** Dry Run' : ''}
                """
                
                echo message
                
                // Send notification (customize as needed)
                // slackSend(
                //     channel: '#deployments',
                //     color: 'good',
                //     message: message
                // )
            }
        }
        
        failure {
            script {
                def message = """
                ❌ **Deployment Failed**
                
                **Application:** myapp
                **Environment:** ${params.ENVIRONMENT}
                **Action:** ${params.ACTION}
                **Image Tag:** ${env.FINAL_IMAGE_TAG}
                **Git Commit:** ${env.GIT_SHORT_COMMIT}
                **Build:** #${env.BUILD_NUMBER}
                **Error:** See build logs for details
                """
                
                echo message
                
                // Send notification (customize as needed)
                // slackSend(
                //     channel: '#deployments',
                //     color: 'danger',
                //     message: message
                // )
            }
        }
        
        cleanup {
            script {
                // Clean workspace
                cleanWs()
            }
        }
    }
}
