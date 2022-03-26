@Library('dynatrace@master') _
@Library('keptn-library@6.0.0')
def keptn = new sh.keptn.Keptn()

def tagMatchRules = [
  [
    meTypes: [
      [meType: 'SERVICE']
    ],
    tags : [
      [context: 'CONTEXTLESS', key: 'app', value: 'carts'],
      [context: 'CONTEXTLESS', key: 'environment', value: 'dev']
    ]
  ]
]



pipeline {
  agent {
    label 'maven'
  }
  
  
	parameters{
         string(defaultValue: 'stage', description: 'Stage of your Keptn project where tests are triggered in', name: 'stage', trim: false) 
         string(defaultValue: '', description: 'Keptn Context ID', name: 'shkeptncontext', trim: false) 
         string(defaultValue: '', description: 'Triggered ID', name: 'triggeredid', trim: false) 
	}
    
  environment {
    APP_NAME = "carts"
    VERSION = readFile('version').trim()
    ARTEFACT_ID = "sockshop/" + "${env.APP_NAME}"
    TAG = "${env.DOCKER_REGISTRY_URL}:5000/library/${env.ARTEFACT_ID}"
    TAG_DEV = "${env.TAG}-${env.VERSION}-${env.BUILD_NUMBER}"
    TAG_STAGING = "${env.TAG}-${env.VERSION}"
    CLASS = "GOLD"
    REMEDIATION = "Ansible"
	  DT_META = "keptn_project=sockshop keptn_service=${env.APP_NAME} keptn_stage=${env.stage} SCM=${env.GIT_URL} Branch=${env.GIT_BRANCH} Version=${env.VERSION} Owner=ace@dynatrace.com FriendlyName=sockshop.carts SERVICE_TYPE=BACKEND Project=sockshop DesignDocument=https://sock-corp.com/stories/${env.APP_NAME} Class=${env.CLASS} Remediation=${env.REMEDIATION}"
  }
  stages {
	  
	  stage('Initialize Keptn') {
		  steps{
			  script{
        		keptn.keptnInit project:"sockshop", service:"carts", stage:"${params.stage}"
			  }
		       }
    }
    stage('Maven build') {
      steps {
        checkout scm
        container('maven') {
          sh 'mvn -B clean package'
        }
      }
    }
    stage('Docker build') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('docker') {
          sh "docker build -t ${env.TAG_DEV} ."
        }
      }
    }
    stage('Docker push to registry'){
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('docker') {
          sh "docker push ${env.TAG_DEV}"
        }
      }
    }
    stage('Deploy to dev namespace') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('kubectl') {
          sh "sed -i 's#image: .*#image: ${env.TAG_DEV}#' manifest/carts.yml"
          sh "sed -i 's#value: \"DT_CUSTOM_PROP_PLACEHOLDER\".*#value: \"${env.DT_META}\"#' manifest/carts.yml"
          sh "kubectl -n dev apply -f manifest/carts.yml"
        }
      }
    }
    stage('DT send deploy event') {
      when {
          expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
          }
      }
      steps {
        container("curl") {
          script {
            def status = pushDynatraceDeploymentEvent (
              tagRule : tagMatchRules,
              customProperties : [
                [key: 'Jenkins Build Number', value: "${env.BUILD_ID}"],
                [key: 'Git commit', value: "${env.GIT_COMMIT}"]
              ]
            )
          }
        }
      }
    }
    stage('Run health check in dev') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        echo "Waiting for the service to start..."
        container('kubectl') {
          script {
            def status = waitForDeployment (
              deploymentName: "${env.APP_NAME}",
              environment: 'dev'
            )
            if(status !=0 ){
              currentBuild.result = 'FAILED'
              error "Deployment did not finish before timeout."
            }
          }
        }

        container('jmeter') {
          script {
            def status = executeJMeter ( 
              scriptName: 'jmeter/basiccheck.jmx', 
              resultsDir: "HealthCheck_${env.APP_NAME}_dev_${env.VERSION}_${BUILD_NUMBER}",
              serverUrl: "${env.APP_NAME}.dev", 
              serverPort: 80,
              checkPath: '/health',
              vuCount: 1,
              loopCount: 1,
              LTN: "HealthCheck_${BUILD_NUMBER}",
              funcValidation: true,
              avgRtValidation: 0
            )
            if (status != 0) {
              currentBuild.result = 'FAILED'
              error "Health check in dev failed."
            }
          }
        }
      }
    }
    stage('Run functional check in dev') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('jmeter') {
          script {
            def status = executeJMeter (
              scriptName: "jmeter/${env.APP_NAME}_load.jmx", 
              resultsDir: "FuncCheck_${env.APP_NAME}_dev_${env.VERSION}_${BUILD_NUMBER}",
              serverUrl: "${env.APP_NAME}.dev", 
              serverPort: 80,
              checkPath: '/health',
              vuCount: 1,
              loopCount: 1,
              LTN: "FuncCheck_${BUILD_NUMBER}",
              funcValidation: true,
              avgRtValidation: 0
            )
            if (status != 0) {
              currentBuild.result = 'FAILED'
              error "Functional check in dev failed."
            }
          }
        }
      }
    }
    
	  
   stage('Send Finished Event Back to Keptn') {
        // Send test.finished Event back
	   steps{
		   script{
        def keptnContext = keptn.sendFinishedEvent eventType: "deployment", keptnContext: "${params.shkeptncontext}", triggeredId: "${params.triggeredid}", result:"pass", status:"succeeded", message:"jenkins tests succeeded"
        //def keptnContext = keptn.sendDeliveryTriggeredEvent image:"${env.TAG_DEV}"
	String keptn_bridge = env.KEPTN_BRIDGE
        echo "Open Keptns Bridge: ${keptn_bridge}/trace/${keptnContext}"
		   }}
    }
	
    stage('Mark artifact for staging namespace') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*'
        }
      }
      steps {
        container('docker'){
          sh "docker tag ${env.TAG_DEV} ${env.TAG_STAGING}"
          sh "docker push ${env.TAG_STAGING}"
        }
      }
    }
    stage('Deploy to staging') {
      when {
        beforeAgent true
        expression {
          return env.BRANCH_NAME ==~ 'release/.*'
        }
      }
      steps {
        build job: "k8s-deploy-staging",
          parameters: [
            string(name: 'APP_NAME', value: "${env.APP_NAME}"),
            string(name: 'TAG_STAGING', value: "${env.TAG_STAGING}"),
            string(name: 'VERSION', value: "${env.VERSION}"),
            string(name: 'DT_CUSTOM_PROP', value: "${env.DT_META}"),
            string(name: 'QUALITYGATE_PROVIDER', value: "Keptn Quality Gates")
          ]
      }
    }
  }
}
