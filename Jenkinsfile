pipeline {
    
 options {
   disableConcurrentBuilds()
   timeout(time: 1, unit: 'HOURS')
   ansiColor('xterm')
 }
 
 parameters {
    choice (name: 'action', choices: 'create\ndestroy', description: 'Create/update or destroy the gke cluster.')
    string (name: 'cluster', defaultValue : 'demo', description: "gke cluster name.")
    string (name: 'gke_version', defaultValue : '1.20.8-gke.2100', description: "gke version.")
    string (name: 'machine_type', defaultValue : 'e2-medium', description: "node machine type.")
    string (name: 'num_nodes', defaultValue : '2', description: "Number of gke nodes.")
    string (name: 'region', defaultValue : 'europe-west4', description: "gcp region.")

    credentials (
      name: 'service_account', 
      credentialType: 'org.jenkinsci.plugins.plaincredentials.impl.FileCredentialsImpl',
      defaultValue: '', 
      description: 'gcloud service account', 
      required: true
    )
 }

 
 agent { label 'master' }
 
 environment {
    // Set path to tool gcloud
    PATH = "${env.PATH}:${env.JENKINS_HOME}/tools/com.cloudbees.jenkins.plugins.gcloudsdk.GCloudInstallation/latest/bin"
  }

 stages {
    stage('Setup') {
      steps {
        tool name: 'latest', type: 'gcloud'
        script {
          cluster = params.cluster
          currentBuild.displayName = "#" + env.BUILD_NUMBER + " " + params.action + " cluster " + cluster

          withCredentials([file(credentialsId: params.service_account, variable: 'GC_KEY')]) {
            sh """
              gcloud version
              gcloud auth activate-service-account --key-file=${GC_KEY}
            """
          }

          echo "Getting the gcp project from the service account and setting it..."
          p = sh(returnStdout: true, script: "gcloud info --format='value(config.account)'").trim()
          p = (p.split('@'))[1]
          project = (p.split("\\."))[0]

          // Then set the project
          sh "gcloud config set project ${project}"
        }
      }
    }

    stage('Deploy') {
      when {
          expression { params.action == 'create' }
      }
      steps {
        script {
          input "Create gke cluster ${params.cluster}?" 
          sh """
            gcloud container clusters create "${params.cluster}" \
              --region "${params.region}" \
              --cluster-version "${params.gke_version}" \
              --machine-type "${params.machine_type}" \
              --num-nodes "${params.num_nodes}"
          """

        }
      }
    }
    
    stage('Destroy') {
      when {
          expression { params.action == 'destroy' }
      }
      steps {
        script {
          input "Delete gke cluster ${params.cluster}?" 
          sh "gcloud container clusters delete ${params.cluster} --region ${params.region} --quiet"
        }
      }
    }
  }
}
