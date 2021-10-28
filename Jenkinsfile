#!/usr/bin/env groovy
// Define variables
GString toolsDir
final String jpVariants = 'jp44,jp43'
final String[] registries = ['gcr.io/axon-pipeline/pipeline']
final def dockerImages = [
        'x86' : 'tensorflow',
        'jp44': 'l4t-tensorflow',
        'jp43': 'l4t-base'
]
final def dockerTags = [
        'x86' : [
                'tf1': '20.09-tf1-py3',
                'tf2': '20.09-tf2-py3'
        ],
        'jp44': [
                'tf1': 'r32.4.4-tf1.15-py3',
                'tf2': 'r32.4.4-tf2.3-py3'
        ],
        'jp43': [
                'tf1': 'r32.3.1',
                'tf2': 'r32.3.1'
        ]
]

final String osType = """ return[
'x86'
]"""
final List jpList = ["\"jp44:selected\"", "\"jp43\""]
final List tfList = ["\"tf1:selected\"", "\"tf2\""]
final List defaultItem = ["\"Not Applicable:disabled:selected\""]
final String jpType = populateType(defaultItem, jpList)
final String tfVersion = populateTF(defaultItem, tfList)




// Properties step to set the Active choice parameters via
// Declarative Scripting
//noinspection GroovyAssignabilityCheck
properties([
        parameters([
                [
                        $class     : 'ChoiceParameter',
                        choiceType : 'PT_SINGLE_SELECT',
                        name       : 'OS',
                        description: 'Pick to osType to build',
                        script     : [
                                $class        : 'GroovyScript',
                                fallbackScript: [
                                        classpath: [],
                                        sandbox  : false,
                                        script   : 'return ["ERROR"]'
                                ],
                                script        : [
                                        classpath: [],
                                        sandbox  : false,
                                        script   : osType
                                ]
                        ]
                ],
                [
                        $class              : 'CascadeChoiceParameter',
                        choiceType          : 'PT_CHECKBOX',
                        name                : 'TF_VERSION',
                        description         : 'Pick tensorflow version(s) to build',
                        referencedParameters: 'JP_TYPE',
                        script              : [
                                $class        : 'GroovyScript',
                                fallbackScript: [
                                        classpath: [],
                                        sandbox  : false,
                                        script   : 'return ["ERROR"]'
                                ],
                                script        : [
                                        classpath: [],
                                        sandbox  : false,
                                        script   : tfVersion
                                ]
                        ]
                ]
        ])
])

pipeline {
    agent any
    parameters {
        choice(name: 'TAG', choices: ['major', 'minor', 'patch'], description: 'Pick the right version')
    }

    environment {
        PATH = "$WORKS"
    }
    options {
        timestamps()
        disableConcurrentBuilds()
        ansiColor('xterm')
//        timeout(time: 24, unit: 'HOURS')
    }
    stages {
        stage('Validate parms') {
            steps {
                script {
                    if (params.TF_VERSION == 'Not Applicable' || params.TF_VERSION == '') {
                        error 'You must select at least one TF version'

                    }
                }
            }
        }

        // stage('Build requirements.txt file') {
        //     steps {
        //         script {
        //             toolsDir = "${env.WORKSPACE}/tools"
        //             sh("""#!/usr/bin/env bash
        //             python3 ${toolsDir}/build_requirements_file.py -d ${toolsDir} -f requirements.txt
        //             """)
        //             final String enum34 = (sh(returnStdout: true,
        //                     script: """#!/usr/bin/env bash
        //                     grep 'enum34' requirements.txt
        //                     """) as String).trim()
        //             sh("""#!/usr/bin/env bash
        //             sed -i '/opencv-python/d' requirements.txt
        //             sed -i '/enum34/d' requirements.txt
        //             echo \"${enum34}\" >> requirements.txt
        //             python3 ${toolsDir}/requirements_cleaner.py -f requirements.txt
        //             """)
        //         }
        //     }
        // }

        // stage('Set gcloud config') {
        //     steps {
        //         script {
        //             final String gcloud = (sh(returnStdout: true, script: """ which gcloud """) as String).trim()
        //             if (gcloud == null) {
        //                 error('Error: gcloud is not installed. Please install it before building an image.')
        //             }
        //         }
        //         sh("""#!/usr/bin/env bash
        //             gcloud config configurations activate axon-pipeline
        //         """)
        //         withCredentials([file(credentialsId: 'automation-ro', variable: 'RO')]) {
        //             sh('gcloud auth activate-service-account --key-file $RO')
        //         }
        //     }
        // }
        stage('Get OpenCV deb files') {
            parallel {
                stage('x86') {
                    when {
                        anyOf {
                            expression { params.OS == 'all' }
                            expression { params.OS == 'x86' }
                        }
                    }
                    steps {
                        downloadOpenCV('x86')
                    }
                }
                stage('jetson') {                                   // not relevant?
                    when {
                        anyOf {
                            expression { params.OS == 'all' }
                            expression { params.OS == 'jetson' }
                        }
                    }
                    steps {
                        downloadOpenCV('jetson')
                    }
                }
            }
        }
        stage("Init") {
            parallel {
                stage('x86') {
                    when {
                        anyOf {
                            expression { params.OS == 'all' }
                            expression { params.OS == 'x86' }
                        }
                    }
                    steps {
                        script {
                            toolsDir = "${env.WORKSPACE}/pl3_ci_tools"
                            final def selectedTF = (params.TF_VERSION as String).split(',')
                            final String version = params.TAG
                            for (final String tfVer : selectedTF) {
                                String tagName = bumpVersion('x86', toolsDir, version, "_${tfVer}")
                                buildFlow('x86',
                                        registries,
                                        tagName,
                                        "${dockerTags['x86'][tfVer]}",
                                        "${tfVer}",
                                        "${dockerImages['x86']}",
                                )
                            }

                        } // end of script
                    } // end of steps x86_tf1
                    post {
                        failure {
                            error "Failed, exiting now..."
                        }
                    }
                } // end of stage x86_tf1
                stage('jetson') {
                    when {
                        anyOf {
                            expression { params.OS == 'all' }
                            expression { params.OS == 'jetson' }
                        }
                    }
                    steps {
                        script {
                            toolsDir = "${env.WORKSPACE}/tools"
                            final def selectedTF = (params.TF_VERSION as String).split(',')
                            final def jpTypes = (params.OS == 'all') ? jpVariants : params.JP_TYPE
                            final def selectedJP = jpTypes.split(',')
                            final String version = params.TAG
                            for (final String jetpack : selectedJP) {
                                for (final String tfVer : selectedTF) {
                                    String tagName = bumpVersion('jetson', toolsDir, version,"_${jetpack}_${tfVer}")
                                    buildFlow('jetson',
                                            registries,
                                            tagName,
                                            "${dockerTags[jetpack][tfVer]}",
                                            "${tfVer}",
                                            "${dockerImages[jetpack]}",
                                            jetpack
                                    )
                                }
                            }
                        }
                    }
                }
            }
        }
    } // end of stages
    post {
        failure {
            error "Failed, exiting now..."
        }
        aborted {
            cleanUp()
        }
        success {
            cleanUp()
        }
    }
}

def bumpVersion(final String ostype, final GString toolsDir, final String version, final String extra = '') {
    String args = "--build=${env.BUILD_NUMBER} -p docker/${ostype}/version${extra}.yml -d 3.0.0-alpha"
    if (version == 'major') {
        args += ' --major'
    } else if (version == 'minor') {
        args += ' --minor'
    }
    return (sh(returnStdout: true, script: """python3 ${toolsDir}/bump_version.py ${args}""") as String).trim()
}

def cleanUp() {
    cleanWs()
    dir("${env.WORKSPACE}@tmp") {
        deleteDir()
    }
    dir("${env.WORKSPACE}@script") {
        deleteDir()
    }
    dir("${env.WORKSPACE}@script@tmp") {
        deleteDir()
    }
    sh("docker images -f 'dangling=true' -q | xargs docker rmi -f")
}

def downloadOpenCV(final String osType) {
    sh("""#!/usr/bin/env bash
            docker pull pachyderm/opencv:latest

    """) // ./docker/${osType}/tmp/ - originally belongs to line 284
}

void buildFlow(final String osType, final String[] registries, final String gitTag, final String dockerTag,
               final String tfVersion, final String dockerImg = '', final String jetpack = '') {

    GString buildArgs = (dockerImg) ?
            "--build-arg IMAGE=${dockerImg} --build-arg TAG=${dockerTag}" :
            "--build-arg TAG=${dockerTag}"

    buildArgs += (jetpack) && (jetpack == 'jp43') ?
            " --build-arg JP3=true --build-arg TF=${tfVersion}" :
            " --build-arg JP3=false"

    final GString remoteTag = (osType == 'x86') ?
            "pl${gitTag}_${tfVersion}" :
            "pl${gitTag}_${jetpack}_${tfVersion}"

    final GString buildType = (osType == 'x86') ?
            "x86_${tfVersion}" :
            "jetson_${jetpack}_${tfVersion}"

    final GString latestTag = (osType == 'x86') ?
            "${tfVersion}-latest" :
            "${jetpack}-${tfVersion}-latest"

    try {
        stage("Build ${buildType}") {
            if (osType == 'jetson') {
                sh("""#!/usr/bin/env bash
                # gsutil -m cp -r 'gs://protobuf-wheel/protobuf-3.8.0-cp36-cp36m-linux_aarch64.whl' ./docker/${osType}/tmp/
                if ! cat /proc/sys/fs/binfmt_misc/qemu-aarch64 | grep -Pw '(?<=flags: )OCF' > /dev/null 2>&1; then
                    docker run --rm --privileged multiarch/qemu-user-static --reset -p yes -c yes
                fi
                """)
            }

            echo "Buiding docker image for ${buildType}"
            sh("""#!/usr/bin/env bash
                exec  > >(tee ${env.WORKSPACE}/output.log) 2> >(tee ${env.WORKSPACE}/error.log >&2)
                docker build -f docker/${osType}/Dockerfile \
                 --force-rm \
                 ${buildArgs} \
                 -t ${osType}:${remoteTag} .
            """)
        } //end stage Build

//         stage("Push ${buildType}") {
//             final String dockerID = (sh(returnStdout: true,
//                     script: """#!/usr/bin/env bash 
// docker images ${osType}:${remoteTag} -q
// """) as String).trim()
//             withCredentials([file(credentialsId: 'automation-rw', variable: 'RW')]) {
//                 sh('''
//                 gcloud config configurations activate default
//                 gcloud auth activate-service-account --key-file $RW
//                 ''')
//            }
        //     for (final String registry in registries) {
        //         sh("""#!/usr/bin/env bash
        //         exec > >(tee ${env.WORKSPACE}/output.log) 2> >(tee ${env.WORKSPACE}/error.log >&2)
        //         docker images ${registry}/${osType}  --format '{{.Repository}}:{{.Tag}}' -q | xargs docker rmi -f
        //         docker tag ${dockerID} ${registry}/${osType}:${remoteTag}
        //         docker tag ${dockerID} ${registry}/${osType}:${latestTag}
        //         docker push ${registry}/${osType} --all-tags
        //         """)
        //         script {
        //             withCredentials([sshUserPrivateKey(credentialsId: 'bitbucket_private_ssh_key', keyFileVariable: 'SSH_KEY')]) {
        //                 sh("""#!/usr/bin/env bash
        //                 git add 'docker/${osType}/version*.yml'
        //                 git commit -m 'Bump ${buildType} pipeline to version ${gitTag}' 'docker/${osType}/version*.yml'
        //                 git push -u origin master
        //                 """)
        //             }
        //         }
        //         sh("""
        //             gcloud config configurations activate axon-pipeline
        //         """)
        //         withCredentials([file(credentialsId: 'automation-ro', variable: 'RO')]) {
        //             sh('gcloud auth activate-service-account --key-file $RO')
        //         }
        //     }
        // } // end of stage push
    } // end of try
    catch (final error) {
        throw error as Throwable
    }
} // end of function