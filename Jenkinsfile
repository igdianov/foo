pipeline {
    agent {
	    kubernetes {
	        // Change the name of jenkins-maven label to be able to use yaml configuration snippet
	        label "maven-jenkins"
	        // Inherit from Jx Maven pod template
	        inheritFrom "maven"
	        // Add scheduling configuration to Jenkins builder pod template
	        yaml """
spec:
  nodeSelector:
    cloud.google.com/gke-preemptible: true

  # It is necessary to add toleration to GKE preemtible pool taint to the pod in order to run it on that node pool
  tolerations:
  - key: gke-preemptible
    operator: Equal
    value: true
    effect: NoSchedule
"""        
	    }    
    }
    environment {
      ORG               = 'igdianov'
      APP_NAME          = 'foo'
      CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')
    }
    stages {
      stage('CI Build and push snapshot') {
        when {
          branch 'PR-*'
        }
        environment {
          PREVIEW_VERSION = "0.0.0-SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER"
          PREVIEW_NAMESPACE = "$APP_NAME-$BRANCH_NAME".toLowerCase()
          HELM_RELEASE = "$PREVIEW_NAMESPACE".toLowerCase()
        }
        steps {
          container('maven') {
            sh "make preview"
          }
        }
      }
      stage('Build Release') {
        when {
          branch 'master'
        }
        steps {
          container('maven') {
            // ensure we're not on a detached head
            sh "make checkout"

            // so we can retrieve the version in later steps
            sh "make version"
            
            // Let's test first
            sh "make verify"

            // Let's make tag in Git
            sh "make tag"
            
            // Let's deploy to Nexus
            sh "make deploy"
          }
        }
      }
      stage('Update Versions') {
        when {
          branch 'master'
        }
        steps {
          container('maven') {
            // Let's push changes and open PRs to downstream repositories
            sh "make updatebot/push-version"

            // Let's update any open PRs
            sh "make updatebot/update"

            // Let's publish release notes in Github using commits between previous and last tags
            sh "make changelog"
          }
        }
      }
    }
    post {
        success {
            cleanWs()
        }
/*
	failure {
            input """Pipeline failed. 
We will keep the build pod around to help you diagnose any failures. 
Select Proceed or Abort to terminate the build pod"""

        }
*/
    }
}
