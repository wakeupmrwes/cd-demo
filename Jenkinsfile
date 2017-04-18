#!/usr/bin/env groovy

//def variants = ['open', 'ee']
def variants = ['ee']
//def channels = ['testing/master', 'EarlyAccess', 'stable']
def channels = ['testing/master']
 

def cluster = [:]
def install = [:]


def get_job_name(channel, variant, build_number) {
  def channel_sanitized = channel.tr('/', '-')
  def job_name = "cd-demo-${variant}-${channel_sanitized}-${build_number}"

  return job_name
}


def get_template_url(variant, channel) {
  def template_url = ''

  if (variant.equals('ee')) {
    template_url = "http://s3.amazonaws.com/downloads.mesosphere.io/dcos-enterprise/${channel}/cloudformation/ee.single-master.cloudformation.json"
  }
  if (variant.equals('open')) {
    template_url = "http://s3.amazonaws.com/downloads.dcos.io/dcos/${channel}/cloudformation/single-master.cloudformation.json"
  }

  return template_url
}


def get_dcos_url(job_name) {
  def dcos_url = sh(
    returnStdout: true,
    script: "./dcos-launch describe -i info-${job_name}.json | jq -r '.masters[0].public_ip'"
  ).trim()

  return dcos_url
}


def get_elb_url(job_name) {
  def elb_url = sh(
    returnStdout: true,
    script: "./dcos-launch describe -i info-${job_name}.json | jq -r '.public_agents[0].public_ip'"
  ).trim()

  return elb_url
}


node('py35') {
  wrap([$class: 'AnsiColorBuildWrapper', 'colorMapName': 'XTerm']) {

    // Git clone
    stage('Git clone') {
      checkout scm
    }


    // Install cd-demo requirements
    stage('Install requirements') {
      sh "pip3 install -r requirements.txt"
      sh "pip3 install httpie"
    }


    // Prepare the execution environment
    stage('Prepare environment') {
      sh "git config --global user.email 'velocity-team@mesosphere.com'"
    }


    // Install dcos-launch binary & requirements
    stage('Install dcos-launch') {
      sh "wget 'https://downloads.dcos.io/dcos/testing/master/dcos-launch' && chmod +x dcos-launch"
      sh "apt-get install -y gettext"
    }


    // Use dcos-launch to acquire a DC/OS cluster
    stage('Acquire DC/OS cluster') {
      withCredentials([[
        $class: 'AmazonWebServicesCredentialsBinding',
        credentialsId: 'Teamcity AWS Development Account',
        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
      ]]) {
        for (v in variants) {
          def variant = v
 
          for (c in channels) {
            def channel = c

            def job_name = get_job_name(channel, variant, env.BUILD_NUMBER)
            def template_url = get_template_url(variant, channel)

            sh """
              envsubst <<EOF > config-${job_name}.yaml
---
launch_config_version: 1
template_url: ${template_url}
deployment_name: ${job_name}
provider: aws
aws_region: eu-central-1
aws_access_key_id: ${env.AWS_ACCESS_KEY_ID}
aws_secret_access_key: ${env.AWS_SECRET_ACCESS_KEY}
template_parameters:
    KeyName: default
    AdminLocation: 0.0.0.0/0
    PublicSlaveInstanceCount: 1
    SlaveInstanceCount: 1
EOF
            """

            cluster[variant + ':' + channel] = {
              sh "./dcos-launch create -c config-${job_name}.yaml -i info-${job_name}.json"
              sh "./dcos-launch wait -i info-${job_name}.json"
            }
          }
        }

        parallel cluster
      }
    }


    // Install Jenkins on test clusters
    stage('Install Jenkins') {
      withCredentials([
        string(credentialsId: 'CD_DEMO_OAUTH_TOKEN', variable: 'CD_DEMO_OAUTH_TOKEN')
      ]) {
        for (v in variants) {
          def variant = v
 
          for (c in channels) {
            def channel = c

            def job_name = get_job_name(channel, variant, env.BUILD_NUMBER)

            install[variant + ':' + channel] = {
              def dcos_url = get_dcos_url(job_name)

              if (variant.equals('ee')) {
                sh "python bin/demo.py install http://${dcos_url}/"
              }
              if (variant.equals('open')) {
                sh "python bin/demo.py install --dcos-oauth-token ${env.CD_DEMO_OAUTH_TOKEN} http://${dcos_url}/"
              }
            }
          }
        }

        parallel install
      }
    }


    // Install Jenkins on test clusters
    stage('Run pipeline demo') {
      withCredentials([
        string(credentialsId: 'CD_DEMO_OAUTH_TOKEN', variable: 'CD_DEMO_OAUTH_TOKEN'),
        string(credentialsId: 'CD_DEMO_DOCKER_PASSWORD', variable: 'CD_DEMO_DOCKER_PASSWORD')
      ]) {
        for (v in variants) {
          def variant = v
 
          for (c in channels) {
            def channel = c

            def job_name = get_job_name(channel, variant, env.BUILD_NUMBER)

            def dcos_url = get_dcos_url(job_name)
            def elb_url = get_elb_url(job_name)

            sshagent(['4ff09dce-407b-41d3-847a-9e6609dd91b8']) {
              sh "git checkout -b ci-${job_name}"
              sh "git push origin ci-${job_name}"

              if (variant.equals('ee')) {
                sh """
                  python bin/demo.py pipeline \
                  --password ${env.CD_DEMO_DOCKER_PASSWORD} \
                  http://${elb_url}/ \
                  http://${dcos_url}/
                """
              }
              if (variant.equals('open')) {
                sh """
                  python bin/demo.py pipeline \
                  --dcos-oauth-token ${env.CD_DEMO_OAUTH_TOKEN} \
                  --password ${env.CD_DEMO_DOCKER_PASSWORD} \
                  http://${elb_url}/ \
                  http://${dcos_url}/
                """
              }
            }

            demo[variant + ':' + channel] = {
              // ...
            }
          }
        }

        parallel demo
      }
    } 

  }
}
