class EnvMap { // define constants 
  private Map contexts = [
    dev: [
      AWS_REGION: 'ap-northeast-2',
      JENKINS_ROLE: 'aicc-dev-role-jenkins-worker',
      TF_WORKSPACE: 'AICC-CONNECT-CORE-IAC-DEV-AP-NORTHEAST-2',
    ],
    stg: [
      AWS_REGION: 'ap-northeast-2',
      JENKINS_ROLE: 'aicc-stg-role-jenkins-worker',
      TF_WORKSPACE: 'AICC-CONNECT-CORE-IAC-STG-AP-NORTHEAST-2',
    ],
    prd: [
      AWS_REGION: 'ap-northeast-2',
      JENKINS_ROLE: 'aicc-prd-role-jenkins-worker',
      TF_WORKSPACE: 'AICC-CONNECT-CORE-IAC-PRD-AP-NORTHEAST-2',
    ],
  ]

  String initMap(final Script script, final String env) {
    contexts.each { context, map ->
      if (env == context) {
        map.each { key, value ->
          script.env."$key" = value
        }
      }
    }

    return "Initialization of Environment variables completed!"
  }
}

// Trunk (master)
final isTrunkBranch() {
  return {
    allOf {
      expression { env.BRANCH == 'master' }
      expression { params.TRIGGER in ['PUSH EVENT', 'MANUAL', 'JENKINS-UI'] }
    }
  }
}

// Next candidate (X.Y.Z-rcN)
final isNextCandidateTag() {
  return {
    allOf {
      expression { env.BRANCH =~ /^\d+\.\d+\.\d+-rc\d+$/ }
      expression { params.TRIGGER in ['REF CREATED', 'MANUAL'] }
    }
  }
}

// Stable candidate (X.Y.Z)
final isStableCandidateTag() {
  return {
    allOf {
      expression { env.BRANCH =~ /^\d+\.\d+\.\d+$/ }
      expression { params.TRIGGER in ['REF CREATED', 'MANUAL'] }
    }
  }
}

// Emergency candidate (X.Y.Z-hfN)
final isEmergencyCandidateTag() {
  return {
    allOf {
      expression { env.BRANCH =~ /^\d+\.\d+\.\d+-hf\d+$/ }
      expression { params.TRIGGER in ['REF CREATED', 'MANUAL'] }
    }
  }
}

final canDeployTo(String targetEnv) {
  if (targetEnv == null || targetEnv.isAllWhitespace()) {
    throw new Error("Unsupported environment code to deploy - cause: your input (empty)")
  }
  switch (targetEnv.toUpperCase()) {
    case 'DEV':
      return { anyOf {
        expression { isTrunkBranch() } // trunk
      }}
    case 'STG':
      return { anyOf {
        expression { isNextCandidateTag() } // Case 1. Next candidate (X.Y.Z-rcN)
        expression { isStableCandidateTag() } // Case 2. Stable candidate (X.Y.Z)
      }}
    case 'PRD':
      return { anyOf {
        expression { isStableCandidateTag() } // Case 1. Stable candidate (X.Y.Z)
        expression { isEmergencyCandidateTag() } // Case 2. Emergency candidate (X.Y.Z)
      }}
    default:
      throw new Error("Unsupported environment code to deploy - cause: your input (" + targetEnv + ")")
  }
}

def printParameters(params) {
    echo 'Parameters:'
    params.each { key, value ->
        echo "${key}: ${value}"
    }
}

pipeline {
    agent {
      node {
        label 'jenkins-jenkins-agent'
      }
    }
    
/*    
    parameters {
        choice(name: 'JAVA_VERSION', choices: ['N/A', '8', '11', '17', '21', 'openjdk-8'], description: 'Select Java version')
        choice(name: 'NODE_VERSION', choices: ['N/A', '12.22.6', '14.17.6', '14.20.0', '16.19.0', '18.16.0', '20.3.1'], description: 'Select Node.js version')
        // 다른 환경변수들도 비슷한 방식으로 정의 가능
    }

  parameters {
    // Git commit variables
    string(name: 'BRANCH', defaultValue: 'main', description: 'the branch name or tag name that triggered the build (without refs/heads/ or refs/tags/)')
    string(name: 'NAME', defaultValue: 'test', description: 'name test')
  }
*/
  environment {
    TARGET_ENV = 'stg'
    TF_INIT_VARS = "${new EnvMap().initMap(this, env.TARGET_ENV)}"
    TF_WORKSPACE_PATH = "./environment/${env.TF_WORKSPACE.toLowerCase()}"
    TF_IN_AUTOMATION = true
  }
    stages {
        stage('Build') {
            steps {
                script {
                    // Print all parameters dynamically
                    echo "TF_INIT_VARS: ${TF_INIT_VARS}"
                    echo "TF_WORKSPACE_PATH: ${TF_WORKSPACE_PATH}"
                    echo "TF_WORKSPACE: ${TF_WORKSPACE}"
                }
            }
        }
    stage("DEV-Deployment") {
      when { allof { expression { canDeployTo('DEV') } } }
      steps {
        echo "================================ ${STAGE_NAME} ================================"
        script {
          deployTo('dev')
        }
      }
    }

    stage("STG-Deployment") {
      when { expression { canDeployTo('STG') } }
      steps {
        echo "================================ ${STAGE_NAME} ================================"
        script {
          applovalReqquest('stg')
          deployTo('stg')
        }
      }
    }

    stage("PRD-Deployment") {
      when { expression { canDeployTo('PRD') } }
      steps {
        echo "================================ ${STAGE_NAME} ================================"
        script {
          applovalReqquest('prd')
          deployTo('prd')
        }
      }
    }      
}
