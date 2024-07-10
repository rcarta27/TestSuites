pipeline {
    agent { 
            node {
                label 'docker-agent-python'
                }
            }
            
            parameters {
            string(name: 'ApplicationScope', defaultValue: '', description: 'Comma-separated list of LifeTime applications to deploy.')
            string(name: 'ApplicationScopeWithTests', defaultValue: '', description: 'Comma-separated list of LifeTime applications to deploy (including test applications)')
            string(name: 'TriggeredBy', defaultValue: 'N/A', description: 'Name of LifeTime user that triggered the pipeline remotely.')
          }
          
        options { skipStagesAfterUnstable() }
            
            environment {
                DevelopmentEnvironment = 'development environment # 1'
                NonProductionEnvironment = 'non-production environment # 2'
                ArtifactsFolder = "Artifacts"
                LifeTimeHostname = 'https://cmiti-lt.outsystemsenterprise.com/'
                LifeTimeAPIVersion = '2'
                AuthorizationToken = credentials('LifeTimeServiceAccountToken')
                BddEnvironmentURL = 'https://cmiti-dev.outsystemsenterprise.com/'
                OSPackageVersion = '0.9.0'
            }
  
 
    stages {
        stage('Install Python Dependencies') {
            steps {
                sh script: "mkdir ${env.ArtifactsFolder}", label: 'Create artifacts folder'
                withEnv(["HOME=${env.ArtifactsFolder}"]) {
                sh script: '''
                        #/bin/bash
                        pip install -q -I virtualenv --user
                        export PATH="$ArtifactsFolder/.local/bin:$PATH"
                '''
                withPythonEnv('python3') {
                      sh script: '''
                                python3 -m pip install --upgrade pip
                                pip install -U outsystems-pipeline
                                ''' 
                      sh script: "pip install -U outsystems-pipeline==\"${env.OSPackageVersion}\"", label: 'Install required packages'
                    }
                }
            }
        }
        stage('Get Latest Tags') {
            steps {
                withEnv(["HOME=${env.ArtifactsFolder}"]) {
                withPythonEnv('python3') {
                    echo "Pipeline run triggered remotely by '${params.TriggeredBy}' for the following applications (including tests): '${params.ApplicationScopeWithTests}'"
                    sh script: "python3 -m outsystems.pipeline.fetch_lifetime_data --artifacts \"${env.ArtifactsFolder}\" --lt_url ${env.LifeTimeHostname} --lt_token ${env.AuthorizationToken} --lt_api_version ${env.LifeTimeAPIVersion}", label: 'Retrieve list of Environments and Applications'
                    // Deploy the application list, with tests, to the Regression environment
                      lock('deployment-plan-REG') {
                        sh script: "python3 -m outsystems.pipeline.deploy_latest_tags_to_target_env --artifacts \"${env.ArtifactsFolder}\" --lt_url ${env.LifeTimeHostname} --lt_token ${env.AuthorizationToken} --lt_api_version ${env.LifeTimeAPIVersion} --source_env \"${env.DevelopmentEnvironment}\" --destination_env \"${env.NonProductionEnvironment}\" --app_list \"${params.ApplicationScopeWithTests}\"", label: "Deploy latest application tags (including tests) to ${env.NonProductionEnvironment}"
                        }
                    }
                }
            }
        }

        stage('Run Regression') {
              when {
                // Checks if there are any test applications in scope before running the regression stage
                expression { return params.ApplicationScope != params.ApplicationScopeWithTests }
              }
                 steps {
                        withPythonEnv('python3') {
                          // Generate the URL endpoints of the BDD tests
                          sh script: "python3 -m outsystems.pipeline.generate_unit_testing_assembly --artifacts \"${env.ArtifactsFolder}\" --app_list \"${params.ApplicationScopeWithTests}\" --cicd_probe_env ${'https://cmiti-dev.outsystemsenterprise.com//BDDFramework/rest/v1/BDDTestRunner/TestApp/HomeScreen'} --bdd_framework_env ${env.BddEnvironmentURL}", label: 'Generate URL endpoints for BDD test suites'
                          // Run those tests and generate a JUnit test report
                          sh script: "python3 -m outsystems.pipeline.evaluate_test_results --artifacts \"${env.ArtifactsFolder}\"", returnStatus: true, label: 'Run BDD test suites and generate JUnit test report'          
                        }
                      }
                      post {
                        always {
                          // Publish results in JUnit test report
                          junit testResults: "${env.ArtifactsFolder}/junit-result.xml", allowEmptyResults: true, skipPublishingChecks: true
                        }
                      }
                    }
        
        stage('Accept changes') {
          steps {
            milestone(ordinal: 40, label: 'before-approval')
            timeout(time:1, unit:'DAYS') {
              input 'Accept changes and deploy to Non-Production?'
            }
            milestone(ordinal: 50, label: 'after-approval')
            }
        }
         stage('Deploy Non-Production') {
          steps {
            withEnv(["HOME=${env.ArtifactsFolder}"]) {
            withPythonEnv('python3') {
              // Deploy the application list to the Production environment
              lock('deployment-plan-REG') {
                            sh script: "python3 -m outsystems.pipeline.deploy_latest_tags_to_target_env --artifacts \"${env.ArtifactsFolder}\" --lt_url ${env.LifeTimeHostname} --lt_token ${env.AuthorizationToken} --lt_api_version ${env.LifeTimeAPIVersion} --source_env \"${env.DevelopmentEnvironment}\" --destination_env \"${env.NonProductionEnvironment}\" --app_list \"${params.ApplicationScopeWithTests}\"", label: "Deploy latest application tags (including tests) to ${env.NonProductionEnvironment}"
                        }
                    }
                }
            }
        }
        stage('Accept changes and deploy to Production') {
          steps {
            milestone(ordinal: 40, label: 'before-approval')
            timeout(time:1, unit:'DAYS') {
              input 'Accept changes and deploy to Production?'
            }
            milestone(ordinal: 50, label: 'after-approval')
            }
        }
         stage('Deploy Production') {
          steps {
            withEnv(["HOME=${env.ArtifactsFolder}"]) {
            withPythonEnv('python3') {
              // Deploy the application list to the Production environment
              lock('deployment-plan-PRD') {
                sh script: "python3 -m outsystems.pipeline.deploy_latest_tags_to_target_env --artifacts \"${env.ArtifactsFolder}\" --lt_url ${env.LifeTimeHostname} --lt_token $env.AuthorizationToken --lt_api_version ${env.LifeTimeAPIVersion} --source_env \"${env.NonProductionEnvironment}\" --destination_env \"${env.ProductionEnvironment}\" --app_list \"${params.ApplicationScope}\" --manifest \"${env.ArtifactsFolder}/deployment_data/deployment_manifest.cache\"", label: "Deploy latest application tags to ${env.ProductionEnvironment}"
                         }
                    }
                }
            }
        }
    }
    post {
        // It will always store the cache files generated, for observability purposes, and notifies the result
        always {
          withEnv(["HOME=${env.ArtifactsFolder}"]) {
          dir ("${env.ArtifactsFolder}") {
            archiveArtifacts artifacts: "**/*.cache"
                }
            }
        }
        failure {
          withEnv(["HOME=${env.ArtifactsFolder}"]) {
          dir ("${env.ArtifactsFolder}") {
            archiveArtifacts artifacts: 'DeploymentConflicts', allowEmptyArchive: true
                }
            }
        }
        cleanup { 
            withEnv(["HOME=${env.ArtifactsFolder}"]) {
            dir ("${env.ArtifactsFolder}") {
            deleteDir()
                }
            }
        }
    }
}
