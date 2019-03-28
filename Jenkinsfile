properties(
    [
        parameters(
            [
                string(description: 'Artifactory server ID', defaultValue: 'artifactory-entplus-us-west', name: 'RT_SERVER_ID'),
                string(description: 'Resolver repo name', defaultValue: 'jcenter', name: 'RT_RESOLVER_REPO'),
                string(description: 'Deployer repo name', defaultValue: 'gradle-dev', name: 'RT_DEPLOYER_REPO'),
                string(description: 'SonarQube URL', defaultValue: 'SonarQube server id', name: 'SONAR_SERVER_ID'),
                string(description: 'Artifactory production repository', defaultValue: '', name: 'RT_PRODUCTION_REPO_NAME'),
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
    node('k8s') {
        def buildInfo
        def server
        def rtGradle
        def rtUrl
        def rtResolverRepoUrl
        def jobName = env.JOB_NAME
        def jobNumber = env.BUILD_NUMBER

        if (! params.RT_SERVER_ID || ! params.RT_RESOLVER_REPO || ! params.RT_DEPLOYER_REPO || ! params.SONAR_SERVER_ID) {
            throw new Exception("Missing required parameter")
        }

        stage('Checkout') {
            checkout scm
            server = Artifactory.server params.RT_SERVER_ID
            rtUrl = server.url
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
                String sonarGradleTasks = "clean sonarqube -PrtResolverRepoUrl=${rtResolverRepoUrl} -Pversion=1.${jobNumber}"
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
                    'targetRepo'         : "${params.RT_PRODUCTION_REPO_NAME}",

                    'buildName'          : buildInfo.name,
                    'buildNumber'        : buildInfo.number,
                    'comment'            : 'Production ready build',
                    'status'             : 'Released',
                    'sourceRepo'         : params.RT_DEPLOYER_REPO,
                    'includeDependencies': true,
                    'copy'               : true,
                    'failFast'           : true
            ]

            server.promote promotionConfig
        }

        stage("Create Release Bundle") {

            rtServiceId = pipelineUtils.restGet("${rtUrl}/api/system/service_id", server.credentialsId)

            def aql = generateAQLQuery(params.RT_PRODUCTION_REPO_NAME, jobName, jobNumber)

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


            res = pipelineUtils.restPost("${distributionUrl}/api/v1/release_bundle", artifactoryCredentialId, releaseBundleBodyJson)
        }

        stage('Distribute release bundle') {
            def distributeReleaseBundleBody = readJSON file: 'distribute-release-bundle-body.json'
            res = pipelineUtils.restPost("${distributionUrl}/api/v1/distribution/${releaseBundleName}/${buildNumber}", artifactoryCredentialId, distributeReleaseBundleBody.toString())

            for (i = 0; true; i++) {
                res = pipelineUtils.restGet("${distributionUrl}/api/v1/release_bundle/${releaseBundleName}/${buildNumber}/distribution", artifactoryCredentialId)

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
