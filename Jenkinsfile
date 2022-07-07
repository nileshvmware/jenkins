pipeline {
 agent {
        label {
            label "DT"
            customWorkspace "/var/lib/jenkins/workspace/${env.JOB_NAME}"
        }
 }
 options {
     skipDefaultCheckout()
    }
 parameters {
  choice(
   name: 'Deploy_Through',
   choices:"Branch\nImage",
   description: "You wanna deploy through Branch / Image!")
  string (
   name: 'BRANCH',
   defaultValue: 'dev',
   description: 'git branch want to deploy')
  string (
   name: 'image_version',
   defaultValue: '${BUILD_DISPLAY_NAME}',
   description: 'Please left as default or pass image')
  choice(
   name: 'Run_Sonar',
   choices:"Yes\nNo",
   description: "Check sonar qualitygate!")
    }
    tools {
        maven "mvn-DT"
        nodejs "node-DT"
    }
 stages {
  stage('Git checkout source code') {
    steps {
     checkout([$class: 'GitSCM',
     branches: [[name: '$BRANCH']],
     doGenerateSubmoduleConfigurations: false,
     extensions: [[$class: 'CleanBeforeCheckout']],
     submoduleCfg: [],
     userRemoteConfigs: [[credentialsId: 'DT', url: 'DT']]
    ])
    script {
     commit_id = sh(returnStdout: true, script: 'git rev-parse --short HEAD')
     commit_id = commit_id.replaceAll(/\s*$/, '')
    }
        }
  }stage("set build name"){
            steps {
                script {
      currentBuild.displayName = "${COUNTRY}-${ENVIRONMENT}-${BUILD_NUMBER}-${commit_id}-${RUN_SONAR}"
      currentBuild.description = "${COUNTRY}-${ENVIRONMENT}-${BUILD_NUMBER}-${commit_id}-${RUN_SONAR}"
                }
            }
  }stage('Git checkout K8s code') {
            steps {
                checkout([$class: 'GitSCM',
     branches: [[name: '*/master']],
     doGenerateSubmoduleConfigurations: false,
     extensions: [[$class: 'CleanCheckout', $class: 'RelativeTargetDirectory', relativeTargetDir: 'devops']],
     submoduleCfg: [],
     userRemoteConfigs: [[credentialsId: 'DT', url: 'DT']]
    ])
            }
        }stage ('Build & Run Test') {
   when {
                expression { params.Deploy_Through == 'Branch' }
            }
   steps {
    sh "mvn -Dmaven.repo.local=/var/lib/jenkins/.m2/repository/DT clean install -f assembly"
   }
  }stage('Sonarqube analysis') {
   when {
    allOf {
     expression { params.Run_Sonar == 'Yes' }
     expression { params.Deploy_Through == 'Branch' }
    }
            }
   environment {
    scannerHome = tool 'sonar'
   }
   steps {
    withSonarQubeEnv('sonar') {
     sh "${scannerHome}/bin/sonar-scanner \
      -Dsonar.projectKey=DT \
      -Dsonar.projectName=DT \
      -Dsonar.projectVersion=DT \
      -Dsonar.sources=. \
      -Dsonar.java.libraries=DT \
      -Dsonar.java.binaries=DT \
      -Dsonar.exclusions=DT \
      "
    }
    timeout(time: 10, unit: 'MINUTES') {
     waitForQualityGate abortPipeline: true
    }
   }
  }stage('jacoco code coverage') {
   when {
                expression { params.Deploy_Through == 'Branch' }
            }
            steps {
    jacoco(
     execPattern: '**/**.exec',
     classPattern: '**/classes',
     sourcePattern: 'src/main/java',
     exclusionPattern: 'src/test*'
    )
            }
        }stage('Image Build and upload to nexus') {
   when {
                expression { params.Deploy_Through == 'Branch' }
            }
   steps {
    sh '''
    imagename=DT
    ./devops/make_image.sh ${imagename} ${image_version}
    '''
   }
  }stage('Deploy Dev?') {
   steps {
    timeout(time: 5, unit: 'MINUTES') {
     input "Do you Want to deploy on dev env?"
     sh '''
     K8sENV=dev
     imagename=DT
     AWS_PROFILE=DT kubectl apply -f devops/pl/${K8sENV}/${imagename}-${K8sENV}.yml
     '''
    }
   }
  }stage('API Health Check') {
   steps {
    timeout(time: 5, unit: 'MINUTES') {
     sh '''
####################
     '''
    }
   }
  }stage('Rollback Dev?') {
   steps {
    timeout(time: 5, unit: 'MINUTES') {
     script {
      env.DT_ROLL_BACK_DEV = input message: 'User input required',
       parameters: [choice(name: 'Rollback', choices: 'no\nyes', description: 'Choose "yes" if you want to Rollback')]
     }
    }
   }
  }stage('RollingBack Dev') {
   when {
   environment(name: 'DT_ROLL_BACK_DEV', value: 'yes')
   }
            steps {
    script {
     skipRemainingStages = true
     println "skipRemainingStages = ${skipRemainingStages}"
    }
    sh '''
    K8sENV=dev
    imagename=DT
    AWS_PROFILE=DT kubectl rollout undo deployment ${imagename} --namespace eshop-poland-${K8sENV}
    '''
            }
        }}post {
        success {
            mail to:"ashok@DT.com", subject:"SUCCESS: ####", body: "#############"
        }
        failure {
            mail to:"ashok@DT.com", subject:"FAILURE: ####", body: "################"
        }
    }}