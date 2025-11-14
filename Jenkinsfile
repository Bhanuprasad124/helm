pipeline {
    agent any
    
    tools {
        nodejs "node18" // Name from Manage Jenkins → Global Tool Configuration
    }

    environment {
        GIT_CREDENTIALS_ID = 'github-auth-token'  // Jenkins GitHub credentials ID
        REPO_URL = 'https://github.com/opshealth/frontend'
    }

    // Parameters for manual triggering
    parameters {
        string(
            name: 'PR_NUMBER',
            defaultValue: '',
            description: 'Pull Request number (e.g., 123). Leave empty if triggered by webhook or Branch Source Plugin. Use this to manually validate a specific PR.'
        )
        string(
            name: 'BRANCH_NAME',
            defaultValue: '',
            description: 'Branch name to build (alternative to PR_NUMBER). Use this for non-PR branches or when PR_NUMBER is not available. Example: feature/my-feature'
        )
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '50'))
        timeout(time: 15, unit: 'MINUTES')
    }

    // This pipeline is designed to be triggered by:
    // 1. GitHub webhook (configure in GitHub repo settings → Webhooks)
    // 2. GitHub Branch Source Plugin (configure in Jenkins job settings)
    // 3. Manual trigger with PR parameters (use PR_NUMBER or BRANCH_NAME parameter)

    stages {
        stage('Checkout PR Branch') {
            steps {
                script {
                    // Determine PR number and branch from various sources
                    def prNumber = null
                    def branchName = null
                    def isManualPR = false
                    
                    // Priority 1: GitHub Branch Source Plugin or Webhook (automated)
                    if (env.CHANGE_ID != null) {
                        prNumber = env.CHANGE_ID
                        branchName = env.CHANGE_BRANCH
                        echo "========================================="
                        echo "✅ Automated PR Trigger"
                        echo "PR #${prNumber}"
                        echo "Branch: ${branchName}"
                        echo "Target: ${env.CHANGE_TARGET}"
                        echo "========================================="
                        
                        // For automated PR triggers, use checkout scm
                        // GitHub Branch Source Plugin automatically configures SCM context
                        echo "Checking out PR using configured SCM..."
                        checkout scm
                    }
                    // Priority 2: Manual PR trigger with PR_NUMBER parameter
                    else if (params.PR_NUMBER != null && params.PR_NUMBER != '') {
                        prNumber = params.PR_NUMBER.trim()
                        isManualPR = true
                        echo "========================================="
                        echo "🔧 Manual PR Trigger"
                        echo "PR #${prNumber}"
                        echo "========================================="
                        
                        // Manual checkout for PR
                        echo "Checking out PR #${prNumber}..."
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: "*/PR-${prNumber}"]],
                            doGenerateSubmoduleConfigurations: false,
                            extensions: [
                                [$class: 'CheckoutOption', timeout: 10],
                                [$class: 'CloneOption', 
                                 depth: 1, 
                                 noTags: false, 
                                 reference: '', 
                                 shallow: true,
                                 timeout: 10]
                            ],
                            submoduleCfg: [],
                            userRemoteConfigs: [[
                                credentialsId: "${env.GIT_CREDENTIALS_ID}",
                                refspec: "+refs/pull/${prNumber}/head:refs/remotes/origin/PR-${prNumber}",
                                url: "${env.REPO_URL}"
                            ]]
                        ])
                        
                        // Set CHANGE_ID for later stages
                        env.CHANGE_ID = prNumber
                    }
                    // Priority 3: Manual trigger with BRANCH_NAME parameter
                    else if (params.BRANCH_NAME != null && params.BRANCH_NAME != '') {
                        branchName = params.BRANCH_NAME.trim()
                        echo "========================================="
                        echo "🔧 Manual Branch Trigger"
                        echo "Branch: ${branchName}"
                        echo "========================================="
                        
                        // Checkout specific branch
                        echo "Checking out branch: ${branchName}..."
                        git credentialsId: "${env.GIT_CREDENTIALS_ID}",
                            url: "${env.REPO_URL}",
                            branch: "${branchName}"
                    }
                    // Priority 4: Fallback to environment variable or default
                    else {
                        branchName = env.BRANCH_NAME ?: 'main'
                        echo "========================================="
                        echo "📦 Default Branch Trigger"
                        echo "Branch: ${branchName}"
                        echo "========================================="
                        
                        // Checkout default branch
                        echo "Checking out branch: ${branchName}..."
                        git credentialsId: "${env.GIT_CREDENTIALS_ID}",
                            url: "${env.REPO_URL}",
                            branch: "${branchName}"
                    }
                    
                    // Display commit info
                    sh '''
                        echo "Current commit:"
                        git log -1 --oneline
                        echo ""
                        echo "Current branch:"
                        git branch --show-current
                    '''
                }
            }
        }

        stage('Verify Node.js') {
            steps {
                sh 'node -v'
                sh 'npm -v'
            }
        }

        stage('Create .npmrc') {
            steps {
                withCredentials([string(credentialsId: 'NPM_RC', variable: 'NPM_RC_TOKEN')]) {
                    sh '''
                        echo "@fortawesome:registry=https://npm.fontawesome.com/" > .npmrc
                        echo "//npm.fontawesome.com/:_authToken=${NPM_RC_TOKEN}" >> .npmrc
                    '''
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Build') {
            steps {
                script {
                    // Set minimal environment variables for build validation
                    // These are defaults - can be customized based on your needs
                   
                    sh '''
                        npm run build
                    '''
                }
            }
        }

        stage('Validate Build Output') {
            steps {
                script {
                    // Verify that build output exists
                    def distExists = fileExists 'dist'
                    if (!distExists) {
                        error("Build failed: dist directory not found. Build may have failed silently.")
                    }
                    
                    // Check if dist directory has content
                    def distFileCount = sh(
                        script: '''
                            if [ ! "$(ls -A dist)" ]; then
                                echo "0"
                                exit 1
                            else
                                ls -1 dist | wc -l
                            fi
                        ''',
                        returnStdout: true
                    ).trim()
                    
                    if (distFileCount == '0' || distFileCount.toInteger() == 0) {
                        error("Build failed: dist directory is empty")
                    }
                    
                    echo "✅ Build validation successful - dist directory contains ${distFileCount} items"
                    
                    // Show sample output
                    sh '''
                        echo "Sample build output:"
                        ls -lh dist/ | head -10
                    '''
                }
            }
        }
    }

    post {
        always {
            // Clean up workspace (optional)
            cleanWs()
        }
        
        success {
            script {
                // Update GitHub PR status to success
                if (env.CHANGE_ID) {
                    echo "✅ Build passed for PR #${env.CHANGE_ID}"
                    // GitHub status update is typically handled by GitHub Branch Source Plugin
                    // Or you can use the GitHub API here if needed
                    try {
                        step([
                            $class: 'GitHubCommitStatusSetter',
                            reposSource: [$class: "ManuallyEnteredRepositorySource", url: "${env.REPO_URL}"],
                            contextSource: [$class: "ManuallyEnteredCommitContextSource", commitShaSource: [$class: "ManuallyEnteredShaSource", sha: env.GIT_COMMIT]],
                            errorHandlers: [],
                            statusResultSource: [
                                $class: "ConditionalStatusResultSource",
                                results: [[$class: "AnyBuildResult", message: "Build passed", state: "SUCCESS"]]
                            ]
                        ])
                    } catch (Exception e) {
                        echo "Note: GitHub status update failed (may require additional plugin configuration): ${e.getMessage()}"
                    }
                }
            }
        }
        
        failure {
            script {
                // Update GitHub PR status to failure
                if (env.CHANGE_ID) {
                    echo "❌ Build failed for PR #${env.CHANGE_ID}"
                    try {
                        step([
                            $class: 'GitHubCommitStatusSetter',
                            reposSource: [$class: "ManuallyEnteredRepositorySource", url: "${env.REPO_URL}"],
                            contextSource: [$class: "ManuallyEnteredCommitContextSource", commitShaSource: [$class: "ManuallyEnteredShaSource", sha: env.GIT_COMMIT]],
                            errorHandlers: [],
                            statusResultSource: [
                                $class: "ConditionalStatusResultSource",
                                results: [[$class: "AnyBuildResult", message: "Build failed - check Jenkins logs", state: "FAILURE"]]
                            ]
                        ])
                    } catch (Exception e) {
                        echo "Note: GitHub status update failed (may require additional plugin configuration): ${e.getMessage()}"
                    }
                }
            }
        }
        
        unstable {
            script {
                if (env.CHANGE_ID) {
                    echo "⚠️ Build unstable for PR #${env.CHANGE_ID}"
                }
            }
        }
    }
}

