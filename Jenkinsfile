properties(
    [
        parameters(
            [
                string(description: 'Artifactory server ID', defaultValue: 'artifactory-entplus-us-west', name: 'RT_SERVER_ID'),
                string(description: 'Resolver repo name', defaultValue: 'jcenter', name: 'RT_RESOLVER_REPO'),
                string(description: 'Deployer repo name', defaultValue: 'gradle-dev', name: 'RT_DEPLOYER_REPO'),
                string(description: 'SonarQube URL', defaultValue: '', name: 'SONAR_URL'),
                string(description: 'SonarQube Project Key', defaultValue: 'gradle-poc', name: 'SONAR_PROJECT'),
                string(description: 'SonarQube Token', defaultValue: '', name: 'SONAR_TOKEN'),
                booleanParam(description: 'deployer repo name', defaultValue: true, name: 'XRAY_FAIL_BUILD'),
            ]
        )

    ]
)

@NonCPS
def readReport(String reportFile) {
    String contents = new File(reportFile).getText('UTF-8')
    Properties properties = new Properties()
    InputStream is = new ByteArrayInputStream(contents.getBytes());
    properties.load(is)
    is.close()
    properties
}

def restGet(String url) {
    def res = httpRequest url: url, consoleLogResponseBody: true
    def httpStatusCode = res.getStatus()
    def respContent = res.getContent()
    if (httpStatusCode >= 400) {
        throw new Exception("HTTP request failed: ${respContent}")
    }
    respContent
}

def getTaskResult(String taskUrl) {
    def res = restGet(taskUrl)
    readJson text: res
}

timestamps {
    node {
        def buildInfo
        def server
        def rtUrl
        def rtGradle

        stage('Checkout') {
            checkout scm
        }

        stage('Build & Deploy') {
            if (! params.RT_SERVER_ID || ! params.RT_RESOLVER_REPO || ! params.RT_DEPLOYER_REPO || ! SONAR_URL || ! SONAR_TOKEN || ! SONAR_PROJECT) {
                throw new Exception("Missing required parameter")
            }

            server = Artifactory.server params.RT_SERVER_ID
            rtGradle = Artifactory.newGradleBuild()
            rtGradle.resolver server: server, repo: params.RT_RESOLVER_REPO
            rtGradle.deployer server: server, repo: params.RT_DEPLOYER_REPO
            def properties = readReport("$WORKSPACE/build/sonar/report-task.txt")
            rtGradle.deployer.addProperty("sonar.ceTaskId", properties.ceTaskId)
            rtGradle.deployer.addProperty("sonar.dashboardUrl", properties.dashboardUrl)
            def taskResult = getTaskResult(properties.ceTaskUrl)
            println taskResult.task.status
            rtGradle.deployer.addProperty("sonar.task.status", taskResult.task.status)



            rtGradle.useWrapper = true
            rtUrl = server.url
            String gradleTasks = "clean artifactoryPublish sonarqube -PrtRepoUrl=${rtUrl}/${params.RT_RESOLVER_REPO} -Dsonar.projectKey=${params.SONAR_PROJECT} -Dsonar.host.url=${params.SONAR_URL} -Dsonar.login=${params.SONAR_TOKEN} -Pversion=1.${env.BUILD_NUMBER}"
            buildInfo = rtGradle.run buildFile: 'build.gradle', tasks: gradleTasks
            buildInfo.env.collect()

            server.publishBuildInfo buildInfo
        }

        stage('Xray Scan') {
            def scanConfig = [
                    'buildName'      : buildInfo.name,
                    'buildNumber'    : buildInfo.number,
                    'failBuild'      : params.XRAY_FAIL_BUILD
            ]
            def scanResult = server.xrayScan scanConfig

            echo scanResult as String
        }

    }
}
