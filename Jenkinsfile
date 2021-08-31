pipeline {
    
 options {
   disableConcurrentBuilds()
   timeout(time: 1, unit: 'HOURS')
   ansiColor('xterm')
 }
 
 parameters {
    choice (name: 'action', choices: 'create\ndestroy', description: 'Create/update or destroy the gke cluster.')
    string (name: 'cluster', defaultValue : 'demo', description: "gke cluster name;eg demo creates cluster named eks-demo.")
    string (name: 'machine_type', defaultValue : 'g1-small', description: "worker node machine type.")
    string (name: 'num_workers', defaultValue : '1', description: "k8s number of worker instances.")

    credentials (
      name: 'service_account', 
      credentialType: 'org.jenkinsci.plugins.plaincredentials.impl.FileCredentialsImpl',
      defaultValue: '', 
      description: 'gcloud service account', 
      required: true
    )

    string (name: 'zone', defaultValue : 'europe-west4-c', description: "gcp zone.")
    string (name: 'gke_version', defaultValue : '1.20.8-gke.2100', description: "gke version.")
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

            echo "Getting the gcp project from the service account and setting it..."
            p = sh(returnStdout: true, script: "gcloud info --format='value(config.account)'").trim()
            p = (p.split('@'))[1]
            project = (p.split("\\."))[0]

            // Then set the project
            sh "gcloud config set project ${project}"
          }
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
              --zone "${params.zone}" \
              --cluster-version "${params.gke_version}" \
              --release-channel "rapid" \
              --machine-type "${params.machine_type}" \
              --num-nodes "${params.num_workers}"
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
          sh "gcloud container clusters delete ${params.cluster} --zone ${params.zone} --quiet"
        }
      }
    }
  }
}
