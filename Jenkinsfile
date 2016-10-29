#!/usr/bin/env groovy

// Random number generator used for generating directory names
Random randomGen = new Random();

// Define some servers and config - TODO: pull these out to something in future (Chef?)
String garageDockerHost = 'tcp://garagepi.local:2376'
Object rPiCWIImage = null
Object rPiHCImage = null
Object rPiGDCImage = null

String jenkinsGitCredentialsId = 'f3266c33-5ce6-45b8-8fdc-48d38dbfa5d6'
String dockerCredentialsId = 'docker-hub-login'

/************/
/* Pipeline */
/************/

/* build: Build, Unit Test, Static Analysis, Package */
stage 'build'

// Build what we can in parallel to optimise build time
parallel(
	rPiCWIBuild: {
		rPiCWIImage = doRPiCWIBuild()
	},
	rPiHCBuild: { rPiHCImage = doRPiDockerImageBuild('ha-rpi-hc') },
	rPiGDCBuild: { rPiGDCImage = doRPiDockerImageBuild('ha-rpi-gdc') },
	failFast: false)

/* auto-test: Provision, Deploy, Automated component, service, integration, acceptance testing */
stage 'auto-test'

// TODO: test

/* prod-deploy: Deploy, Smoke Test */
// Only 1 build can deploy to the servers at a time
stage name: 'prod-deploy', concurrency: 1

// Deploy what we can in parallel
parallel(
	rPiCWIDeploy: {
		doRPiCWIDeploy(rPiCWIImage, garageDockerHost)
	},
	rPiHCDeploy: { doRPiGPIODeviceDockerDeploy(rPiHCImage) },
	rPiGDCDeploy: { doRPiGPIODeviceDockerDeploy(rPiGDCImage) },
	failFast: false)

/*****************/
/* FUNCTION DEFS */
/*****************/

// Raspberry Pi Cam Web Interface builder
def doRPiCWIBuild(String jenkinsGitCredentialsId, String dockerCredentialsId) {
	node('rasbpi') {
		// Note: This has to be done on the Rasberry Pi build slave

		// Check out docker image build script and code to add to it
		dir('RPi_Cam_Web_Interface') {
			git url: 'https://github.com/silvanmelchior/RPi_Cam_Web_Interface.git'
		}
		dir('ha-rpi-cwi') {
			git credentialsId: jenkinsGitCredentialsId, url: 'https://github.com/rcjcooke/ha-rpi-cwi.git'
			sh 'mv ../RPi_Cam_Web_Interface RPi_Cam_Web_Interface'
		}
		// TODO: Bundle release notes + update Wiki

		// Build the docker image
		dir('ha-rpi-cwi') {
			// Build the docker image
			def newRPiCWIBuild = buildAndPushDockerFileToDockerHub("rcjcooke/ha-rpi-cwi:${env.BUILD_TAG}", dockerCredentialsId)
			// Return the image so we can deploy it later
			return newRPiCWIBuild
		}
	}
}

// Raspberry Pi Cam Web Interface Deployer
def doRPiCWIDeploy(Object dockerImage, String deployHost) {
	node('rasbpi') {
		// TODO: Pull these out to config management on a per server basis?
		// withEnv(["DOCKER_HOST=${deployHost}",'DOCKER_TLS_VERIFY=0']) {
			dockerImage.run('-p 80:80 --device /dev/vchiq')
		// }
	}
}

def doRPiDockerImageBuild(String imageName) {
	return checkoutBuildAndPushDockerHubImageFromGitOnNode(
		'raspbi',
		jenkinsGitCredentialsId,
		'https://github.com/rcjcooke/' + imageName + '.git',
		dockerCredentialsId,
		'rcjcooke/' + imageName + ':' + env.BUILD_TAG,
		)
}

// General docker image Raspberry Pi Docker image deployer
def doRPiGPIODeviceDockerDeploy(Object dockerImage) {
	node('rasbpi') {
		dockerImage.run('--device /dev/ttyAMA0:/dev/ttyAMA0 --device /dev/mem:/dev/mem --privileged')
	}
}

/*********************/
/* Utility Functions */
/*********************/
def checkoutBuildAndPushDockerHubImageFromGitOnNode(String nodeName, String jenkinsGitCredentialsId, String gitUrl, String dockerCredentialsId, String dockerTag) {
	node(nodeName) {
		// Check out docker image build script
		def buildDir = getRandomDirName();
		dir(buildDir) {
			git credentialsId: jenkinsGitCredentialsId, url: gitUrl
			// Build the docker image
			def newBuild = buildAndPushDockerFileToDockerHub(dockerTag, dockerCredentialsId)
			// Return the image so we can deploy it later
			return newBuild
		}
	}
}

def buildAndPushDockerFileToDockerHub(String tag, String dockerCredentialsId) {
	// Build the docker image
	def newImage = docker.build tag
	// Docker push (with credentials for Docker Hub)
	withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: dockerCredentialsId, passwordVariable: 'DOCKER_HUB_PASS', usernameVariable: 'DOCKER_HUB_USER']]) {
		// Login to the Docker Hub
		sh 'docker login -u $DOCKER_HUB_USER -p $DOCKER_HUB_PASS'
		// Publish the image to the docker artefact repository
		newImage.push 'latest'
	}
	// Return the image so we can deploy it later
	return newImage
}

def getRandomDirName() {
	def dirName = env.BUILD_TAG + '_' + (Math.abs(randomGen.nextInt() % 10000) + 1)
	return dirName
}
