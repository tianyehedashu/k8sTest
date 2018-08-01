#!/usr/bin/groovy
def label = "mypod-${UUID.randomUUID().toString()}"
podTemplate(label: label, serviceAccount: 'jenkins',
        containers: [
                containerTemplate(
                        name: 'cectest',
                        image: '10.233.14.38:80/fabric8/cec-test2:0.0.1',
                        command: '/bin/sh -c',
                        args: 'cat',
                        ttyEnabled: true,
                        workingDir: '/home/jenkins/'),
                containerTemplate(
                        name: 'clients',
                        image: "fabric8/builder-clients:v703b6d9",
                        command: '/bin/sh -c',
                        args: 'cat',
                        privileged: true,
                        workingDir: '/home/jenkins/',
                        ttyEnabled: true),
                containerTemplate(
                        name: 'compile',
                        image: "10.233.14.38:80/fabric8/cec:725c0a6",
                        command: '/bin/sh -c',
                        args: 'cat',
                        privileged: true,
                        workingDir: '/home/jenkins/',
                        ttyEnabled: true)
        ],
        volumes: [
                secretVolume(secretName: 'jenkins-docker-cfg', mountPath: '/home/jenkins/.docker'),
                secretVolume(secretName: 'jenkins-hub-api-token', mountPath: '/home/jenkins/.apitoken'),
                secretVolume(secretName: 'jenkins-ssh-config', mountPath: '/root/.ssh'),
                secretVolume(secretName: 'jenkins-git-ssh', mountPath: '/root/.ssh-git'),
                hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')
        ]) {
    node(label) {
        try {
            def namespace ='fabric8-staging'
            def rc = ''
            dir('cecProduct') { // switch to subdir
                checkout scm
                stage 'build new docker images'
                echo 'NOTE: running pipelines for the first time will take longer as build and base docker images are pulled onto the node'
                container('compile'){
                  dir('coreservice/frontend') {
                     sh 'pwd'
                     sh 'cdt2 self-update'
                     sh "rm -rf ~/.m2/*"
                     sh 'rm -rf .cdt'
                     sh 'chmod 777 -R buildTools/*'
                     sh 'buildTools/build.sh -c  ALL -d ../backend/webroot'
                  }
                }
                container('clients') {
                    newVersion = sh(script: 'git rev-parse --short HEAD', returnStdout: true).toString().trim()
                    def newImageName = "${env.FABRIC8_DOCKER_REGISTRY_SERVICE_HOST}:${env.FABRIC8_DOCKER_REGISTRY_SERVICE_PORT}/${namespace}/${env.JOB_NAME}:${newVersion}"
                    sh "cd coreservice;docker build -t ${newImageName} . "  
                    sh "docker push ${newImageName}"
                    newImageName = "${env.FABRIC8_DOCKER_REGISTRY_SERVICE_HOST}:${env.FABRIC8_DOCKER_REGISTRY_SERVICE_PORT}/${namespace}/enm.${env.JOB_NAME}:${newVersion}"
                    sh "cd enmservice;docker build -t ${newImageName} . "  
                    sh "docker push ${newImageName}"
                    newImageName = "${env.FABRIC8_DOCKER_REGISTRY_SERVICE_HOST}:${env.FABRIC8_DOCKER_REGISTRY_SERVICE_PORT}/${namespace}/cell.${env.JOB_NAME}:${newVersion}"
                    sh "cd cellinfoservice;docker build -t ${newImageName} . "  
                    sh "docker push ${newImageName}"
                    newImageName = "${env.FABRIC8_DOCKER_REGISTRY_SERVICE_HOST}:${env.FABRIC8_DOCKER_REGISTRY_SERVICE_PORT}/${namespace}/simulator.${env.JOB_NAME}:${newVersion}"
                    sh "cd simulatorservice;docker build -t ${newImageName} . "  
                    sh "docker push ${newImageName}"

                    stage 'generate kubernetes resources'
                    rc = getK8sYaml( {
                        port = 8585
                        label = 'cec'
                        icon = 'https://cdn.rawgit.com/fabric8io/fabric8/dc05040/website/src/images/logos/nodejs.svg'
                        resourceName = 'cec'
                    },newVersion)
                }
                stage 'Rollout Staging'
                kubernetesApply(file: rc, environment: 'fabric8-staging')
            }

            dir('cectest') {
                stage 'verify'
                stage 'Rollout product'
                kubernetesApply(file: rc, environment: 'fabric8-production')
            }
        } catch (exc) {
        emailext body: 'Check console output at $BUILD_URL to view the results.', subject: '$PROJECT_NAME - Build # $BUILD_NUMBER - failed', to: '$DEFAULT_RECIPIENTS'
            throw exc
        }
    }
}

