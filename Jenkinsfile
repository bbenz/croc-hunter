#!/usr/bin/groovy

// load pipeline functions
// Requires pipeline-github-lib plugin to load library from github

@Library('github.com/jldeen/jenkins-pipeline@dev')

def pipeline = new io.estrado.Pipeline()

podTemplate(label: 'jenkins-pipeline', containers: [
    containerTemplate(name: 'jnlp', image: 'jenkinsci/jnlp-slave:3.19-1-alpine', args: '${computer.jnlpmac} ${computer.name}', workingDir: '/home/jenkins', resourceRequestCpu: '200m', resourceLimitCpu: '300m', resourceRequestMemory: '256Mi', resourceLimitMemory: '512Mi'),
    containerTemplate(name: 'docker', image: 'docker:latest', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'golang', image: 'golang:1.8.3', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'helm', image: 'lachlanevenson/k8s-helm:v2.13.0', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl:v1.10.7', command: 'cat', ttyEnabled: true)
],
volumes:[
    hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
]){

  node ('jenkins-pipeline') {

    def pwd = pwd()
    def chart_dir = "${pwd}/charts/croc-hunter"

    checkout scm

    // read in required jenkins workflow config values
    def inputFile = readFile('Jenkinsfile-bb.json')
    def config = new groovy.json.JsonSlurperClassic().parseText(inputFile)
    println "pipeline config ==> ${config}"

    // continue only if pipeline enabled
    if (!config.pipeline.enabled) {
        println "pipeline disabled"
        return
    }

    // set additional git envvars for image tagging
    pipeline.gitEnvVars()

    // If pipeline debugging enabled
    if (config.pipeline.debug) {
      println "DEBUG ENABLED"
      sh "env | sort"

      println "Runing kubectl/helm tests"
      container('kubectl') {
        pipeline.kubectlTest()
      }
      container('helm') {
        pipeline.helmConfig()
      }
    }

    def acct = pipeline.getContainerRepoAcct(config)

    // tag image with version, and branch-commit_id
    def image_tags_map = pipeline.getContainerTags(config)

    // compile tag list
    def image_tags_list = pipeline.getMapValues(image_tags_map)

    stage ('compile and test') {

      container('golang') {
        sh "go test -v -race ./..."
        sh "make bootstrap build"
      }
    }

    stage ('test deployment') {

      container('helm') {

        // run helm chart linter
        pipeline.helmLint(chart_dir)

        // run dry-run helm chart installation
        pipeline.helmDeploy(
          dry_run       : true,
          name          : config.app.name,
          namespace     : config.app.name,
          chart_dir     : chart_dir,
          set           : [
            "imageTag": image_tags_list.get(0),
            "replicas": config.app.replicas,
            "cpu": config.app.cpu,
            "memory": config.app.memory,
            "ingress.hostname": config.app.hostname,
            "imagePullSecrets.name": config.k8s_secret.name,
            "imagePullSecrets.repository": config.container_repo.host,
            "imagePullSecrets.username": env.USERNAME,
            "imagePullSecrets.password": env.PASSWORD,
            "imagePullSecrets.email": "ServicePrincipal@AzureRM",
          ]
        )

      }
    }

    stage ('publish container') {

      container('docker') {

        // perform docker login to container registry as the docker-pipeline-plugin doesn't work with the next auth json format
        withCredentials([[$class          : 'UsernamePasswordMultiBinding', credentialsId: config.container_repo.jenkins_creds_id,
                        usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
          sh "echo ${env.PASSWORD} | docker login -u ${env.USERNAME} --password-stdin ${config.container_repo.host}"
        }

        // build and publish container
        pipeline.containerBuildPub(
            dockerfile: config.container_repo.dockerfile,
            host      : config.container_repo.host,
            acct      : acct,
            repo      : config.container_repo.repo,
            tags      : image_tags_list,
            auth_id   : config.container_repo.jenkins_creds_id,
            image_scanning: config.container_repo.image_scanning
        )
      }

    }

    if (env.BRANCH_NAME =~ "PR-*" ) {
      stage ('deploy to k8s') {
        // Deploy using Helm chart
        container('helm') {
                    // Create secret from Jenkins credentials manager
          withCredentials([[$class          : 'UsernamePasswordMultiBinding', credentialsId: config.container_repo.jenkins_creds_id,
                        usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
          pipeline.helmDeploy(
            dry_run       : false,
            name          : env.BRANCH_NAME.toLowerCase(),
            namespace     : env.BRANCH_NAME.toLowerCase(),
            chart_dir     : chart_dir,
            set           : [
              "imageTag": image_tags_list.get(0),
              "replicas": config.app.replicas,
              "cpu": config.app.cpu,
              "memory": config.app.memory,
              "ingress.hostname": config.app.hostname,
              "imagePullSecrets.name": config.k8s_secret.name,
              "imagePullSecrets.repository": config.container_repo.host,
              "imagePullSecrets.username": env.USERNAME,
              "imagePullSecrets.password": env.PASSWORD,
              "imagePullSecrets.email": "ServicePrincipal@AzureRM",
            ]
          )
          } 
          //  Run helm tests
          if (config.app.test) {
            pipeline.helmTest(
              name        : env.BRANCH_NAME.toLowerCase()
            )
          }

          // delete test deployment
          pipeline.helmDelete(
              name       : env.BRANCH_NAME.toLowerCase()
          )
        }
      }
    }

    // deploy only the master branch
    if (env.BRANCH_NAME == 'master') {
      stage ('deploy to k8s') {
          // Deploy using Helm chart
        container('helm') {
                    // Create secret from Jenkins credentials manager
          withCredentials([[$class          : 'UsernamePasswordMultiBinding', credentialsId: config.container_repo.jenkins_creds_id,
                        usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
          pipeline.helmDeploy(
            dry_run       : false,
            name          : config.app.name,
            namespace     : config.app.name,
            chart_dir     : chart_dir,
            set           : [
              "imageTag": image_tags_list.get(0),
              "replicas": config.app.replicas,
              "cpu": config.app.cpu,
              "memory": config.app.memory,
              "ingress.hostname": config.app.hostname,
              "imagePullSecrets.name": config.k8s_secret.name,
              "imagePullSecrets.repository": config.container_repo.host,
              "imagePullSecrets.username": env.USERNAME,
              "imagePullSecrets.password": env.PASSWORD,
              "imagePullSecrets.email": "ServicePrincipal@AzureRM",
            ]
          )
          
            //  Run helm tests
            if (config.app.test) {
              pipeline.helmTest(
                name          : config.app.name
              )
            }
          }
        }
      }
    }
  }
}