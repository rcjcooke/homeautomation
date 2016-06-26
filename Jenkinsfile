// Define some servers and config - TODO: pull these out to something in future (Chef?)

// TODO: Update this when I install Docker Engine on Rasbpi
def garageDockerHost = "tcp://garagepi.local:2376"


/************/
/* Pipeline */
/************/

stage 'build'

// Build what we can in parallel to optimise build time
parallel(rPiCWIBuild: {
	doRPiCWIBuild()
})

stage 'auto-test'

// TODO: deploy, test

// Only 1 build can deploy to the servers at a time
stage name: 'prod-deploy', concurrency: 1

// Deploy what we can in parallel
parallel(rPiDeploy: {
	doRPiCWIDeploy()
})






/*****************/
/* FUNCTION DEFS */
/*****************/

// Raspberry Pi Cam Web Interface builder
def doRPiCWIBuild() {
	node('rasbpi') {
		// Note: This has to be done on the Rasberry Pi build slave

		// Check out docker image build script and code to add to it
		dir('RPi_Cam_Web_Interface') {
			git url: 'https://github.com/silvanmelchior/RPi_Cam_Web_Interface.git'
		}
		dir('ha-rpi-cwi') {
			git credentialsId: 'f3266c33-5ce6-45b8-8fdc-48d38dbfa5d6', url: 'https://github.com/rcjcooke/ha-rpi-cwi.git'
			sh 'mv ../RPi_Cam_Web_Interface RPi_Cam_Web_Interface'
		}
		// TODO: Bundle release notes + update Wiki

		dir('ha-rpi-cwi') {
			// Build the docker image
			def newRPiCWIBuild = docker.build "rcjcooke/ha-rpi-cwi:${env.BUILD_TAG}"
			// Docker push (with credentials for Docker Hub)
			withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'docker-hub-login', passwordVariable: 'DOCKER_HUB_PASS', usernameVariable: 'DOCKER_HUB_USER']]) {
				// Login to the Docker Hub
				sh 'docker login -u $DOCKER_HUB_USER -p $DOCKER_HUB_PASS'
				// Publish the image to the docker artefact repository
				newRPiCWIBuild.push 'latest'
			}
		}

	}
}

def doRPiCWIDeploy() {
	node('rasbpi') {
		// TODO: Pull these out to config management on a per server basis?
		withEnv(['DOCKER_HOST=${garageDockerHost}','DOCKER_TLS_VERIFY=0']) {
			sh 'docker run -d -p 80:80 rcjcooke/ha-rip-cwi:${env.BUILD_TAG}'
		}
	}
}
