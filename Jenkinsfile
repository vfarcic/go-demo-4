pipeline {
    agent {
        kubernetes {
            label "builder-pod-${UUID.randomUUID().toString()}"
            cloud "go-demo-4-build"
            defaultContainer 'jnlp'
            serviceAccount "build"
            yamlFile "KubernetesPod.yaml"
        }
    }

    environment {
        project="go-demo-4"
        image="vfarcic/go-demo-4"
        domain = "acme.com"
        cmAddr = "cm.acme.com"

        rsaKey="go-demo-rsa-key"
        githubToken="github_token"
    }


    options {
        buildDiscarder logRotator(numToKeepStr: '5')
        disableConcurrentBuilds()
    }

    stages {
        stage('build') {
            steps {
                ciPrettyBuildNumber()

                container('git') {
                    ciBuildEnvVars()
                }

                container('docker') {
                    ciK8sBuildImage(env.image, false, env.BUILD_TAG)
                }
            }
        }

        stage("func-test") {
            steps {
                container("helm") {
                    ciK8sUpgradeBeta(env.project, env.domain, env.BUILD_TAG)
                }

                container("kubectl") {
                    ciK8sRolloutBeta(env.project)
                }

                container("golang") {
                    ciK8sFuncTestGolang(env.project, env.domain)
                }
            }

            post {
                failure {
                    container("helm") {
                        ciK8sDeleteBeta(env.project)
                    }
                }
            }
        }


        stage("Release") {
            when {
                anyOf {
                    branch "master"
                    branch "hotfix-*"
                }
            }

            steps {
                script {
                    container("git") {
                        ciWithGitKey(env.rsaKey) {
                            env.RELEASE_TAG = ciSuggestVersion(ciVersionRead())
                        }
                    }

                    container("helm") {
                        ciContinueAfterTimeout(5, 'MINUTES') {
                            ciConditionalInputExecution(
                                    id: "Release Gate",
                                    message: "Release ${env.project} ?",
                                    ok: "ok",
                                    name: "release") {
                                //docker tag
                                container('docker') {
                                    ciRetag(env.BUILD_TAG, false, ["latest", env.shortGitCommit, env.RELEASE_TAG])
                                }

                                //git tag
                                container('git') {
                                    ciWithGitKey(env.rsaKey) {
                                        ciTagGitRelease(tag: env.RELEASE_TAG)
                                    }
                                }

                                container('gren') {
                                    //https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/
                                    withCredentials([string(credentialsId: env.githubToken, variable: 'TOKEN')]) {
                                        sh "gren release --token=${TOKEN}"
                                    }
                                }

                                container("helm") {
                                    ciK8sPushHelm(env.project, env.RELEASE_TAG, env.cmAddr, true)
                                }
                            }
                        }

                    }

                }
            }
        }
    }

    post {
        always {
            ciWhenNotReleaseBranches {
                container("helm") {
                    ciK8sDeleteBeta(env.project)
                }
            }
        }
    }
}