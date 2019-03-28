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

def restGet(String url) {
    def res = httpRequest url: url, consoleLogResponseBody: true
    def httpStatusCode = res.getStatus()
    def respContent = res.getContent()
    if (httpStatusCode >= 400) {
        throw new Exception("HTTP request failed: ${respContent}")
    }
    respContent
}

Properties getProperties(filename) {
    def properties = new Properties()
    properties.load(new StringReader(readFile(filename)))
    return properties
}

@NonCPS
def jsonParse(text) {
    return new groovy.json.JsonSlurperClassic().parseText(text)
}

timestamps {
    node('k8s') {
        def buildInfo
        def server
        def rtGradle
        def rtResolverRepoUrl

        if (! params.RT_SERVER_ID || ! params.RT_RESOLVER_REPO || ! params.RT_DEPLOYER_REPO || ! params.SONAR_SERVER_ID) {
            throw new Exception("Missing required parameter")
        }

        stage('Checkout') {
            checkout scm
            server = Artifactory.server params.RT_SERVER_ID
            def rtUrl = server.url
            rtResolverRepoUrl = "${rtUrl}/${params.RT_RESOLVER_REPO}"
        }

        stage('Prepare Gradle Environment') {
            rtGradle = Artifactory.newGradleBuild()
            rtGradle.resolver server: server, repo: params.RT_RESOLVER_REPO
            rtGradle.deployer server: server, repo: params.RT_DEPLOYER_REPO
            rtGradle.useWrapper = true
        }

        stage('SonarQube scan') {
            withSonarQubeEnv(params.SONAR_SERVER_ID) {
                String sonarGradleTasks = "clean sonarqube -PrtResolverRepoUrl=${rtResolverRepoUrl} -Pversion=1.${env.BUILD_NUMBER}"
                buildInfo = rtGradle.run buildFile: 'build.gradle', tasks: sonarGradleTasks
            }

            timeout(time: 5, unit: 'MINUTES') {
                def qg = waitForQualityGate()
                if (qg.status != 'OK') {
                    error "Pipeline aborted due to quality gate failure: ${qg.status}"
                }
            }

            properties = getProperties("${WORKSPACE}/build/sonar/report-task.txt")
            ceTaskReport = restGet(properties.ceTaskUrl)
            ceTaskJson = jsonParse(ceTaskReport)
            rtGradle.deployer.addProperty("sonar.ceTaskId", properties.ceTaskId)
            rtGradle.deployer.addProperty("sonar.dashboardUrl", properties.dashboardUrl)
            rtGradle.deployer.addProperty("sonar.task.status", ceTaskJson.task.status)

        }

        stage('Build & Deploy') {
            String publishGradleTasks = "clean artifactoryPublish -PrtResolverRepoUrl=${rtResolverRepoUrl} -Pversion=1.${env.BUILD_NUMBER}"
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
