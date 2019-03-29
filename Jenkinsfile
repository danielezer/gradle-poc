properties(
    [
        parameters(
            [
                string(description: 'Jenkins node label', defaultValue: 'pipes-default', name: 'JENKINS_NODE_LABEL'),
                string(description: 'Artifactory server ID', defaultValue: 'artifactory-azure', name: 'RT_SERVER_ID'),
                credentials(description: 'Artifactory credentials ID', defaultValue: 'artifactory-azure', name: 'RT_CREDENTIALS_ID'),
                string(description: 'Resolver repo name', defaultValue: 'jcenter', name: 'RT_RESOLVER_REPO'),
                string(description: 'Deployer repo name', defaultValue: 'poc-gradle-dev-local', name: 'RT_DEPLOYER_REPO'),
                string(description: 'Artifactory production repository', defaultValue: 'poc-gradle-prod-local', name: 'RT_PRODUCTION_REPO'),
                string(description: 'SonarQube server id', defaultValue: 'sonarqube-azure', name: 'SONAR_SERVER_ID'),
                string(description: 'Distribution service url', defaultValue: '', name: 'DISTRIBUTION_URL'),
                booleanParam(description: 'deployer repo name', defaultValue: true, name: 'XRAY_FAIL_BUILD'),
            ]
        )

    ]
)

def checkHttpResponseCode(httpResponse) {
    def httpStatusCode = httpResponse.getStatus()
    def respContent = httpResponse.getContent()
    if (httpStatusCode >= 400) {
        throw new Exception("HTTP request failed: ${respContent}")
    }
    return respContent
}

def restGet(url, credentialsId) {
    def res = httpRequest url: url, consoleLogResponseBody: true, authentication: credentialsId
    def resContent = checkHttpResponseCode(res)
    resContent
}

def restPost(url, credentialId, body, contentType = 'APPLICATION_JSON') {
    def res = httpRequest url: url, contentType: contentType, authentication: credentialId, httpMode: 'POST', requestBody: body, consoleLogResponseBody: true
    def resContent = checkHttpResponseCode(res)
    resContent
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

def generateAQLQuery(repoName, buildName, buildNumber) {
    aqlQuery = """
            items.find({
                \"\$and\": [
                        {
                            \"repo\": {
                            \"\$match\": \"${repoName}\"
                        }
                        },
                        {
                            \"\$or\": [
                                {
                                    \"@build.name\": \"${buildName}\"
                                }
                        ]
                        },
                        {
                            \"@build.number\": \"${buildNumber}\"
                        }
                ]
            })
            """.replaceAll(" ", "").replaceAll("\n", "")
}

timestamps {
    node(params.JENKINS_NODE_LABEL) {
        def jobName = env.JOB_NAME
        def jobNumber = env.BUILD_NUMBER
        def rtServerId = params.RT_SERVER_ID
        def sonarServerId = params.SONAR_SERVER_ID
        def server = Artifactory.server rtServerId
        def artifactoryCredentialsId = params.RT_CREDENTIALS_ID
        def rtUrl = server.url
        def rtProductionRepo = params.RT_PRODUCTION_REPO
        def rtDeployerRepo = params.RT_DEPLOYER_REPO
        def rtResolverRepo = params.RT_RESOLVER_REPO
        def rtResolverRepoUrl = "${rtUrl}/${rtResolverRepo}"
        def distributionUrl = params.DISTRIBUTION_URL
        def rtGradle
        def buildInfo

        if (! rtServerId || ! rtResolverRepo || ! rtDeployerRepo || ! sonarServerId || ! rtProductionRepo) {
            throw new Exception("Missing required parameter")
        }

        stage('Checkout') {
            checkout scm
        }

        stage('Prepare Build Environment') {
            rtResolverRepoUrl = "${rtUrl}/${rtResolverRepo}"
            rtGradle = Artifactory.newGradleBuild()
            rtGradle.resolver server: server, repo: rtResolverRepo
            rtGradle.deployer server: server, repo: rtDeployerRepo
            rtGradle.useWrapper = true
        }

        stage('SonarQube scan') {
            withSonarQubeEnv(sonarServerId) {
                String sonarGradleTasks = "clean test sonarRunner -PrtResolverRepoUrl=${rtResolverRepoUrl} -Pversion=1.${jobNumber}"
                buildInfo = rtGradle.run buildFile: 'build.gradle', tasks: sonarGradleTasks
            }

            timeout(time: 5, unit: 'MINUTES') {
                def qg = waitForQualityGate()
                if (qg.status != 'OK') {
                    error "Pipeline aborted due to quality gate failure: ${qg.status}"
                }
            }

            properties = getProperties("${WORKSPACE}/build/sonar/report-task.txt")
            ceTaskReport = restGet(properties.ceTaskUrl, null)
            ceTaskJson = jsonParse(ceTaskReport)
            rtGradle.deployer.addProperty("sonar.ceTaskId", properties.ceTaskId)
            rtGradle.deployer.addProperty("sonar.dashboardUrl", properties.dashboardUrl)
            rtGradle.deployer.addProperty("sonar.task.status", ceTaskJson.task.status)

        }

        stage('Build & Deploy') {
            String publishGradleTasks = "clean artifactoryPublish -PrtResolverRepoUrl=${rtResolverRepoUrl} -Pversion=1.${jobNumber}"
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

        stage('Promote Build') {
            def promotionConfig = [
                    'targetRepo'         : rtProductionRepo,
                    'buildName'          : buildInfo.name,
                    'buildNumber'        : buildInfo.number,
                    'comment'            : 'Production ready build',
                    'status'             : 'Released',
                    'sourceRepo'         : rtDeployerRepo,
                    'includeDependencies': true,
                    'copy'               : true,
                    'failFast'           : true
            ]

            println promotionConfig

            server.promote promotionConfig
        }

        stage("Create Release Bundle") {

            def rtServiceId = restGet("${rtUrl}/api/system/service_id", artifactoryCredentialsId)

            def aql = generateAQLQuery(rtProductionRepo, jobName, jobNumber)

            def releaseBundleBody = [
                    'name': "${jobName}",
                    'version': "1.${jobNumber}",
                    'dry_run': false,
                    'sign_immediately': true,
                    'description': 'Release bundle for the example java-project',
                    'spec': [
                            'source_artifactory_id': "${rtServiceId}",
                            'queries': [
                                    [
                                            'aql': "${aql}",
                                            'query_name': 'aql-query'
                                    ]
                            ]
                    ]
            ]

            releaseBundleBodyJson = JsonOutput.toJson(releaseBundleBody)


            restPost("${distributionUrl}/api/v1/release_bundle", artifactoryCredentialsId, releaseBundleBodyJson)
        }

        stage('Distribute release bundle') {
            def distributeReleaseBundleBody = readJSON file: 'distribute-release-bundle-body.json'
            restPost("${distributionUrl}/api/v1/distribution/${releaseBundleName}/${buildNumber}", artifactoryCredentialsId, distributeReleaseBundleBody.toString())

            for (i = 0; true; i++) {
                def res = restGet("${distributionUrl}/api/v1/release_bundle/${releaseBundleName}/${buildNumber}/distribution", artifactoryCredentialsId)

                def jsonResult = readJSON text: res
                def distributionStatus = jsonResult.status.unique()
                distributionStatus = distributionStatus.collect { it.toUpperCase() }
                println "Current status:  ${distributionStatus}"

                if (distributionStatus == ['COMPLETED']) {
                    println "Distribution finished successfully"
                    break
                } else if (distributionStatus.contains('FAILED')) {
                    error("Distribution failed. Response body: ${jsonResult}")
                } else if (i >= 30) {
                    error("Timed out waiting for distribution to complete")
                }

                sleep 2

            }
        }
    }
}
