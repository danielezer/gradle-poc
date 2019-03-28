properties(
    [
        parameters(
            [
                string(description: 'Artifactory server ID', defaultValue: 'artifactory-entplus-us-west', name: 'RT_SERVER_ID'),
                string(description: 'Resolver repo name', defaultValue: 'jcenter', name: 'RT_RESOLVER_REPO'),
                string(description: 'Deployer repo name', defaultValue: 'gradle-dev', name: 'RT_DEPLOYER_REPO'),
                string(description: 'SonarQube URL', defaultValue: '', name: 'SONAR_SERVER_ID'),
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
    readJSON text: res
}

timestamps {
    node('k8s') {
        def buildInfo
        def server
        def rtUrl
        def rtGradle

        if (! params.RT_SERVER_ID || ! params.RT_RESOLVER_REPO || ! params.RT_DEPLOYER_REPO || ! params.SONAR_SERVER_ID) {
            throw new Exception("Missing required parameter")
        }

        stage('Checkout') {
            checkout scm
        }

        stage('Build & Deploy') {
            server = Artifactory.server params.RT_SERVER_ID
            rtGradle = Artifactory.newGradleBuild()
            rtGradle.resolver server: server, repo: params.RT_RESOLVER_REPO
            rtGradle.deployer server: server, repo: params.RT_DEPLOYER_REPO
            rtGradle.useWrapper = true
            rtUrl = server.url
            withSonarQubeEnv(params.SONAR_SERVER_ID) {
                String sonarGradleTasks = "clean sonarqube -PrtRepoUrl=${rtUrl}/${params.RT_RESOLVER_REPO} -Pversion=1.${env.BUILD_NUMBER}"
                buildInfo = rtGradle.run buildFile: 'build.gradle', tasks: sonarGradleTasks
            }

            timeout(time: 5, unit: 'minutes') {
                def qg = waitForQualityGate()
                if (qg.status != 'OK') {
                    error "Pipeline aborted due to quality gate failure: ${qg.status}"
                }
            }

            println qg.taskId


            rtGradle.deployer.addProperty("sonar.ceTaskId", properties.ceTaskId)
            rtGradle.deployer.addProperty("sonar.dashboardUrl", properties.dashboardUrl)
            rtGradle.deployer.addProperty("sonar.task.status", taskResult.task.status)

            String publishGradleTasks = "clean artifactoryPublish -PrtRepoUrl=${rtUrl}/${params.RT_RESOLVER_REPO} -Pversion=1.${env.BUILD_NUMBER}"
            buildInfo = rtGradle.run buildFile: 'build.gradle', tasks: publishGradleTasks
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
