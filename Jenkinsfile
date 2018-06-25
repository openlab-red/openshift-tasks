#!groovy

node('maven') {
    // def DEV_PROJECT="dev"
    // def PROD_PROJECT="prod"
    def mvnCmd = "mvn -s ./nexus_openshift_settings.xml"

    stage('Checkout Source') {
        checkout scm
    }

    def pom = readMavenPom file: 'pom.xml'

    def groupId = pom.groupId
    def artifactId = pom.artifactId
    def version = pom.version

    stage('Build war') {
        echo "Building version ${version}"

        sh "${mvnCmd} clean package -DskipTests"
    }
    
    stage('Unit Tests and Code Analysis') {
      parallel (
            'Test': {
              sh "${mvnCmd} test"
            }, 
        'Static Analysis': {
                sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube:9000 -DskipTests=true -Dsonar.projectName=${JOB_BASE_NAME}"
              }
      )
    }

    stage('Publish to Nexus') {
        sh "${mvnCmd} deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://nexus3:8081/repository/releases"

    }

    stage('Build OpenShift Image') {
        sh "rm -rf oc-build && mkdir -p oc-build/deployments"
        sh "cp target/openshift-tasks.war oc-build/deployments/ROOT.war"
        // clean up. keep the image stream
        sh "oc delete bc,dc,svc,route -l app=tasks -n ${DEV_PROJECT}"
        // create build. override the exit code since it complains about exising imagestream
        sh "oc new-build --name=tasks --image-stream=jboss-eap70-openshift:1.5 --binary=true --labels=app=tasks -n ${DEV_PROJECT} || true"
        // build image
        sh "oc start-build tasks --from-dir=oc-build --wait=true -n ${DEV_PROJECT}"
    }

    stage('Deploy to Dev') {
        sh "oc new-app tasks:latest -n ${DEV_PROJECT}"
        sh "oc expose svc/tasks -n ${DEV_PROJECT}"

        openshiftVerifyDeployment depCfg: 'tasks', namespace: "${DEV_PROJECT}", replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: '', waitUnit: 'sec'

    }

    stage('Integration Test') {
        // TBD: Proper test
        //sh "curl -i -u 'redhat:redhat1!' -H 'Content-Length: 0' -X POST http://tasks:8080/ws/tasks/task1"

        def newTag = "ProdReady-${version}"

        echo "New Tag: ${newTag}"

        openshiftTag alias: 'false', destStream: 'tasks', destTag: newTag, destinationNamespace: "${DEV_PROJECT}", namespace: "${DEV_PROJECT}", srcStream: 'tasks', srcTag: 'latest', verbose: 'false'
    }

    // Blue/Green Deployment into Production
    // -------------------------------------
    def dest = "tasks-green"
    def active = ""

    stage('Prep Production Deployment') {
        sh "oc project ${PROD_PROJECT}"
        sh "oc get route tasks -n ${PROD_PROJECT} -o jsonpath='{ .spec.to.name }' > activesvc.txt"
        active = readFile('activesvc.txt').trim()
        if (active == "tasks-green") {
            dest = "tasks-blue"
        }
        echo "Active svc: " + active
        echo "Dest svc:   " + dest
    }

    stage('Deploy new Version') {
        echo "Deploying to ${dest}"

        sh "oc patch dc ${dest} --patch '{\"spec\": { \"triggers\": [ { \"type\": \"ImageChange\", \"imageChangeParams\": { \"containerNames\": [ \"$dest\" ], \"from\": { \"kind\": \"ImageStreamTag\", \"namespace\": \"${DEV_PROJECT}\", \"name\": \"tasks:ProdReady-$version\"}}}]}}' -n ${PROD_PROJECT}"

        openshiftDeploy depCfg: "${dest}", namespace: "${PROD_PROJECT}", verbose: 'false', waitTime: '', waitUnit: 'sec'

        openshiftVerifyDeployment depCfg: "${dest}", namespace: "${PROD_PROJECT}", replicaCount: '1', verbose: 'false', verifyReplicaCount: 'true', waitTime: '', waitUnit: 'sec'

    }
    stage('Switch over to new Version') {
        input "Switch Production?"

        sh "oc set route-backends tasks -n ${PROD_PROJECT} ${dest}=1"
        sh "oc get route tasks -n ${PROD_PROJECT} > oc_out.txt"
        oc_out = readFile('oc_out.txt')
        echo "Current route configuration: " + oc_out
    }
}
