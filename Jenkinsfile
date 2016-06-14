stage 'build'

// Raspberry Pi Cam Web Interface builder
node 'rasbpi' {
	// This has to be done on the Rasberry Pi build slave
	git url: 'https://github.com/silvanmelchior/RPi_Cam_Web_Interface.git'
	git url: 'https://github.com/silvanmelchior/RPi_Cam_Web_Interface.git'
	
	def newRPiCWIBuild = docker.build "rcjcooke/RPi_CWI:${env.BUILD_TAG}"
	
	// TODO: Bundle release notes + update Wiki
	
	newRPiCWIBuild.push(...) // Store in docker artefact repo
	
}


stage 'auto-test'


stage 'prod-deploy'






node {
    checkout scm
    servers = load 'servers.groovy'
    mvn '-o clean package'
    dir('target') {stash name: 'war', includes: 'x.war'}
}


    