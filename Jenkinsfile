properties(
    [
        parameters(
            [
                string(description: 'Artifactory server ID', defaultValue: 'artifactory-entplus-us-west', name: 'RT_SERVER_ID'),
                string(description: 'Resolver repo name', defaultValue: 'jcenter', name: 'RT_RESOLVER_REPO'),
                string(description: 'Deployer repo name', defaultValue: 'gradle-dev', name: 'RT_DEPLOYER_REPO'),
                string(description: 'SonarQube URL', defaultValue: '', name: 'SONAR_URL'),
                string(description: 'SonarQube Project Key', defaultValue: 'test-gradle', name: 'SONAR_PROJECT'),
                string(description: 'SonarQube Token', defaultValue: '', name: 'SONAR_TOKEN'),
                booleanParam(description: 'deployer repo name', defaultValue: true, name: 'XRAY_FAIL_BUILD'),
            ]
        )

    ]
)

timestamps {
    node {
        def buildInfo
        def server
        def rtUrl
        stage('Checkout') {
            checkout scm
        }

        stage('Build & Deploy') {
            if (! params.RT_SERVER_ID || ! params.RT_RESOLVER_REPO || ! params.RT_DEPLOYER_REPO || ! SONAR_URL || ! SONAR_TOKEN || ! SONAR_PROJECT) {
                throw new Exception("Missing required parameter")
            }

            server = Artifactory.server params.RT_SERVER_ID
            def rtGradle = Artifactory.newGradleBuild()
            rtGradle.resolver server: server, repo: params.RT_RESOLVER_REPO
            rtGradle.deployer server: server, repo: params.RT_DEPLOYER_REPO
            rtGradle.useWrapper = true
            rtUrl = server.url
            String gradleTasks = "clean artifactoryPublish -PrtRepoUrl=${rtUrl}/${params.RT_RESOLVER_REPO}"
            buildInfo = rtGradle.run buildFile: 'build.gradle', tasks: gradleTasks
            buildInfo.env.collect()
            server.publishBuildInfo buildInfo
        }

        stage('Sonar scan') {
            sh "./gradlew --no-daemon sonarqube -PrtRepoUrl=${rtUrl}/${params.RT_RESOLVER_REPO} -Dsonar.projectKey=${params.SONAR_PROJECT_KEY} -Dsonar.host.url=${params.SONAR_URL} -Dsonar.login=${params.SONAR_TOKEN}"
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