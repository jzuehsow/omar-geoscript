properties([
    parameters ([
        string(name: 'BUILD_NODE', defaultValue: 'omar-build', description: 'The build node to run on'),
        booleanParam(name: 'CLEAN_WORKSPACE', defaultValue: true, description: 'Clean the workspace at the end of the run')
    ]),
    pipelineTriggers([
            [$class: "GitHubPushTrigger"]
    ]),
    [$class: 'GithubProjectProperty', displayName: '', projectUrlStr: 'https://github.com/ossimlabs/omar-geoscript'],
    buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '3', daysToKeepStr: '', numToKeepStr: '20')),
    disableConcurrentBuilds()
])

node("${BUILD_NODE}"){

    stage("Checkout branch $BRANCH_NAME")
    {
        checkout(scm)
    }

    stage("Load Variables")
    {
        withCredentials([string(credentialsId: 'o2-artifact-project', variable: 'o2ArtifactProject')]) {
            step ([$class: "CopyArtifact",
                projectName: o2ArtifactProject,
                filter: "common-variables.groovy",
                flatten: true])
        }

        load "common-variables.groovy"
    }

    stage ("Assemble") {
        sh """
        ./gradlew assemble \
            -PossimMavenProxy=${OSSIM_MAVEN_PROXY}
        """
        archiveArtifacts "plugins/*/build/libs/*.jar"
        archiveArtifacts "apps/*/build/libs/*.jar"
    }
    stage("Fortify SCA"){
      steps {
          sh """
             export PATH=${PATH}:/opt/HPE_Security/Fortify_SCA_and_Apps_17.20/bin
             echo hello world 43

             """
          }

    }

    stage ("Publish Nexus")
    {
        withCredentials([[$class: 'UsernamePasswordMultiBinding',
                        credentialsId: 'nexusCredentials',
                        usernameVariable: 'MAVEN_REPO_USERNAME',
                        passwordVariable: 'MAVEN_REPO_PASSWORD']])
        {
            sh """
            ./gradlew publish \
                -PossimMavenProxy=${OSSIM_MAVEN_PROXY}
            """
        }
    }

    stage ("Publish Docker App")
    {
        withCredentials([[$class: 'UsernamePasswordMultiBinding',
                        credentialsId: 'dockerCredentials',
                        usernameVariable: 'DOCKER_REGISTRY_USERNAME',
                        passwordVariable: 'DOCKER_REGISTRY_PASSWORD']])
        {
            // Run all tasks on the app. This includes pushing to OpenShift and S3.
            sh """
            ./gradlew pushDockerImage \
                -PossimMavenProxy=${OSSIM_MAVEN_PROXY}
            """
        }
    }

    try {
        stage ("OpenShift Tag Image")
        {
            withCredentials([[$class: 'UsernamePasswordMultiBinding',
                            credentialsId: 'openshiftCredentials',
                            usernameVariable: 'OPENSHIFT_USERNAME',
                            passwordVariable: 'OPENSHIFT_PASSWORD']])
            {
                // Run all tasks on the app. This includes pushing to OpenShift and S3.
                sh """
                    ./gradlew openshiftTagImage \
                        -PossimMavenProxy=${OSSIM_MAVEN_PROXY}

                """
            }
        }
    } catch (e) {
        echo e.toString()
    }

    stage("Clean Workspace")
    {
        if ("${CLEAN_WORKSPACE}" == "true")
            step([$class: 'WsCleanup'])
    }
}
