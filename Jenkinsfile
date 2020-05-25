@Library('dynatrace@master') _

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

def dashboardTileRules = [
    [
        name : 'Service health', tileType : 'SERVICES', configured : true, filterConfig: null, chartVisible: true,
        bounds : [ top: 38, left : 0, width: 304, height: 304 ],
        tileFilter : [ timeframe : null, managementZone : null ]
            
    ],
    [
        name : 'Custom chart', tileType : 'CUSTOM_CHARTING', configured : true, chartVisible: true,
        bounds : [ top: 38, left : 342, width: 494, height: 304 ],
        tileFilter : [ timeframe : null, managementZone : null ],
        filterConfig : [ type : 'MIXED', customName: 'Response time', defaultName: 'Custom chart', 
            chartConfig : [
                legendShown : true, type : 'TIMESERIES', resultMetadata : [:],
                series : [
                    [ metric: 'builtin:service.response.time', aggregation: 'AVG', percentile: null, type : 'BAR', entityType : 'SERVICE', dimensions : [], sortAscending : false, sortColumn : true, aggregationRate : 'TOTAL' ]
                ],
            ],
        filtersPerEntityType: [:]
        ]
    ],
    [
        name : 'Custom chart', tileType : 'CUSTOM_CHARTING', configured : true, chartVisible: true,
        bounds : [ top: 38, left : 874, width: 494, height: 304 ],
        tileFilter : [ timeframe : null, managementZone : null ],
        filterConfig : [ type : 'MIXED', customName: 'Failure rate (any  errors)', defaultName: 'Custom chart', 
            chartConfig : [
                legendShown : true, type : 'TIMESERIES', resultMetadata : [:],
                series : [
                    [ metric: 'builtin:service.errors.total.rate', aggregation: 'AVG', percentile: null, type : 'BAR', entityType : 'SERVICE', dimensions : [], sortAscending : false, sortColumn : true, aggregationRate : 'TOTAL' ]
                ],
            ],
        filtersPerEntityType: [:]
        ]
    ]
]

def mzContitions = [
    [
        key: [ attribute: 'SERVICE_TAGS' ],
        comparisonInfo: [ type: 'TAG', operator: 'EQUALS',
            value: [ context: 'ENVIRONMENT', key: 'product', value: 'sockshop' ],
            negate: false
        ]
    ],
    [
        key: [ attribute: 'PROCESS_GROUP_PREDEFINED_METADATA', dynamicKey: 'KUBERNETES_NAMESPACE', type: 'PROCESS_PREDEFINED_METADATA_KEY' ],
        comparisonInfo: [ type: 'STRING', operator: 'EQUALS', value: 'dev', negate: false, caseSensitive: false ]
    ]
]

pipeline {
  agent {
    label 'maven'
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
    DT_META = "keptn_project=sockshop-qg keptn_service=${env.APP_NAME} keptn_stage=hardening SCM=${env.GIT_URL} Branch=${env.GIT_BRANCH} Version=${env.VERSION} Owner=ace@dynatrace.com FriendlyName=sockshop.carts SERVICE_TYPE=BACKEND Project=sockshop DesignDocument=https://sock-corp.com/stories/${env.APP_NAME} Class=${env.CLASS} Remediation=${env.REMEDIATION}"
  }
  stages {
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
    stage('DT create synthetic monitor') {
      when {
          expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
          }
      }
      steps {
        container("kubectl") {
          script {
            // Get IP of service
            env.SERVICE_IP = sh(script: 'kubectl get svc ${APP_NAME} -n dev -o \'jsonpath={..status.loadBalancer.ingress..ip}\'', , returnStdout: true).trim()
          }
        }
        container("curl") {
          script {
            def status = dt_createUpdateSyntheticTest (
              testName : "sockshop.dev.${env.APP_NAME}",
              url : "http://${SERVICE_IP}/items",
              method : "GET",
              location : "${env.DT_SYNTHETIC_LOCATION_ID}"
            )
          }
        }
      }
    }
    stage('DT create application detection rule') {
      when {
          expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
          }
      }
      steps {
          container("curl") {
          script {
            def status = dt_createUpdateAppDetectionRule (
              dtAppName : "sockshop.dev.${env.APP_NAME}",
              pattern : "http://${SERVICE_IP}",
              applicationMatchType: "CONTAINS",
              applicationMatchTarget: "URL"
            )
          }
        }
      }
    }
    stage('DT create management zone') {
      when {
          expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
          }
      }
      steps {
        container("curl") {
          script {
            def (int status, String dt_mngtZoneId) = dt_createUpdateManagementZone (
                managementZoneName : 'Sockshop dev',
                ruleType : 'SERVICE',
                managementZoneConditions : mzContitions,
            )
            DT_MGMTZONEID = dt_mngtZoneId
          }
        }
      }
    }
    stage('DT create dashboard') {
      when {
          expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
          }
      }
      steps {
        container("curl") {
          script {
            def status = dt_createUpdateDashboard (
              dashboardName : 'sockshop-dev',
              dashboardManagementZoneName : 'sockshop-dev',
              dashboardManagementZoneId : "${DT_MGMTZONEID}",
              dashboardShared : false,
              dashboardLinkShared : true,
              dashboardPublished : false,
              dashboardTimeframe : '-30m',
              dtDashboardTiles : dashboardTileRules
            )
          }
        }
     }
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
