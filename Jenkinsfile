pipeline {
    agent none

    environment {
        BUILD_CONTEXT = "build-context-${BUILD_ID}.tar.gz"
	PROJECT = "${JENKINS_TEST_PROJECT}"
	GCR_IMAGE = "gcr.io/${PROJECT}/cloudbees-day-seattle-demo:${BUILD_ID}"
	APP_JAR = "cloudbees-day-seattle-demo.jar"
	STAGING_CLUSTER = "staging"
	STAGING_CLUSTER_ZONE = "us-central1-c"
	PROD_CLUSTER = "prod"
	PROD_CLUSTER_ZONE = "us-central1-c"
    }

    stages {
        stage("Build and test") {
	    agent {
    	    	  kubernetes {
      		      cloud 'kubernetes'
      		      label 'mavenpod'
      		      yamlFile 'jenkins/maven-pod.yaml'
		}
	    }
	    steps {
	    	  container('maven') {
		      // build
	    	      sh "mvn clean package"

		      // run tests
		      sh "mvn verify"
		      
		      // archive the build context for kaniko
		      sh "cp target/cloudbees-days-seattle-demo-*.jar $APP_JAR"
		      sh "tar --exclude='./.git' -zcvf /tmp/$BUILD_CONTEXT ."
		      sh "mv /tmp/$BUILD_CONTEXT ."
		      step([$class: 'ClassicUploadStep', credentialsId: env.JENKINS_TEST_CRED_ID, bucket: "gs://${JENKINS_TEST_BUCKET}", pattern: env.BUILD_CONTEXT])    
		  }
	    }
	}
	stage("Publish Image") {
            agent {
    	    	  kubernetes {
      		      cloud 'kubernetes'
      		      label 'kaniko-pod'
      		      yamlFile 'jenkins/kaniko-pod.yaml'
		}
	    }
	    environment {
                PATH = "/busybox:/kaniko:$PATH"
      	    }
	    steps {
	        container(name: 'kaniko', shell: '/busybox/sh') {
		    sh '''#!/busybox/sh
		    /kaniko/executor -f `pwd`/Dockerfile -c `pwd` --context="gs://${JENKINS_TEST_BUCKET}/${BUILD_CONTEXT}" --destination="${GCR_IMAGE}" --build-arg JAR_FILE="${APP_JAR}"
		    '''
		}
	    }
	}
	stage("Deploy to staging") {
            agent {
    	        kubernetes {
      		    cloud 'kubernetes'
      		    label 'gke-deploy'
		    yamlFile 'jenkins/gke-deploy-pod.yaml'
		}
            }
	    steps{
		container('gke-deploy') {
		    sh "sed -i s#IMAGE#${GCR_IMAGE}#g kubernetes/manifest.yaml"
                    step([$class: 'KubernetesEngineBuilder', projectId: env.PROJECT, clusterName: env.STAGING_CLUSTER, zone: env.STAGING_CLUSTER_ZONE, manifestPattern: 'kubernetes/manifest.yaml', credentialsId: env.JENKINS_TEST_CRED_ID, verifyDeployments: true])
		}
            }
	}
        stage('Wait for SRE Approval') {
         steps{
           timeout(time:12, unit:'HOURS') {
              input message:'Approve deployment?'
           }
         }
        }
	stage("Deploy to prod") {
            agent {
    	        kubernetes {
      		    cloud 'kubernetes'
      		    label 'gke-deploy'
		    yamlFile 'jenkins/gke-deploy-pod.yaml'
		}
            }
	    steps{
		container('gke-deploy') {
		    sh "sed -i s#IMAGE#${GCR_IMAGE}#g kubernetes/manifest.yaml"
                    step([$class: 'KubernetesEngineBuilder', projectId: env.PROJECT, clusterName: env.PROD_CLUSTER, zone: env.PROD_CLUSTER_ZONE, manifestPattern: 'kubernetes/manifest.yaml', credentialsId: env.JENKINS_TEST_CRED_ID, verifyDeployments: true])
		}
            }
	}
    }
}