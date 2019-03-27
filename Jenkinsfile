properties(
    [
        parameters(
            [
                string(description: 'Artifactory server ID', defaultValue: 'artifactory-entplus-us-west', name: 'RT_SERVER_ID'),
                string(description: 'resolver repo name', defaultValue: 'jcenter', name: 'RT_RESOLVER_REPO'),
                string(description: 'deployer repo name', defaultValue: 'gradle-dev', name: 'RT_DEPLOYER_REPO'),
                booleanParam(description: 'deployer repo name', defaultValue: true, name: 'XRAY_FAIL_BUILD'),
            ]
        )

    ]
)

timestamps {
    node {
        def buildInfo
        def server
        stage('Checkout') {
            checkout scm
        }

        stage('Build & Deploy') {
            if (! params.RT_SERVER_ID || ! params.RT_RESOLVER_REPO || ! params.RT_DEPLOYER_REPO) {
                throw new Exception("Missing required parameter")
            }

            server = Artifactory.server params.RT_SERVER_ID
            def rtGradle = Artifactory.newGradleBuild()
            rtGradle.resolver server: server, repo: params.RT_RESOLVER_REPO
            rtGradle.deployer server: server, repo: params.RT_DEPLOYER_REPO
            rtGradle.useWrapper = true
            buildInfo = rtGradle.run buildFile: 'build.gradle', tasks: 'clean artifactoryPublish'
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