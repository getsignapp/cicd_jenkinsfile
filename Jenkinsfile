#!groovy

node {

    def SF_CONSUMER_KEY = env.SF_CONSUMER_KEY
    def SF_USERNAME = env.SF_USERNAME
    def SERVER_KEY_CREDENTIALS_ID = env.SERVER_KEY_CREDENTIALS_ID
    def TEST_LEVEL = env.TEST_LEVEL
    def SF_INSTANCE_URL = env.SF_INSTANCE_URL ?: "https://login.salesforce.com"
    def SC_URL = env.SC_URL
	
    def DEPLOYDIR='src/deploy'
    def MAINSOURCEDIR='src/force-app/main/default'
    def PACKAGESOURCEDIR='src/manifest/package.xml'
    
    def toolbelt = tool 'toolbelt'

    // -------------------------------------------------------------------------
    // Check out code from source control.
    // -------------------------------------------------------------------------

    stage('checkout source') {
        checkout scm
	    
	checkout changelog: false, poll: false, scm: [$class: 'GitSCM', branches: [[name: '*/master']], extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'src']], userRemoteConfigs: [[credentialsId: 'github', url: '${SC_URL}']]]
	    
	command "mkdir ${DEPLOYDIR}"
	command "copy ${PACKAGESOURCEDIR} ${DEPLOYDIR}"
	command "xcopy ${MAINSOURCEDIR} ${DEPLOYDIR} /O /X /E /H /K"
    }


    // -------------------------------------------------------------------------
    // Run all the enclosed stages with access to the Salesforce
    // JWT key credentials.
    // -------------------------------------------------------------------------

 	withEnv(["HOME=${env.WORKSPACE}"]) {	
	
	    withCredentials([file(credentialsId: SERVER_KEY_CREDENTIALS_ID, variable: 'server_key_file')]) {
		// -------------------------------------------------------------------------
		// Authenticate to Salesforce using the server key.
		// -------------------------------------------------------------------------

		stage('Authorize to Salesforce') {
			rc = command """"${toolbelt}/sfdx" force:auth:jwt:grant --instanceurl ${SF_INSTANCE_URL} --clientid ${SF_CONSUMER_KEY} --jwtkeyfile ${server_key_file} --username ${SF_USERNAME} --setalias UAT"""
		    if (rc != 0) {
			error 'Salesforce org authorization failed.'
		    }
		}


		// -------------------------------------------------------------------------
		// Deploy metadata and execute unit tests.
		// -------------------------------------------------------------------------

		stage('Deploy and Run Tests') {
		    rc = command """"${toolbelt}/sfdx" force:mdapi:deploy --wait 10 --deploydir ${DEPLOYDIR} --targetusername UAT --testlevel ${TEST_LEVEL}"""
		    if (rc != 0) {
			error 'Salesforce deploy and test run failed.'
		    }
		}


		// -------------------------------------------------------------------------
		// Example shows how to run a check-only deploy.
		// -------------------------------------------------------------------------

		//stage('Check Only Deploy') {
		//    rc = command "${toolbelt}/sfdx force:mdapi:deploy --checkonly --wait 10 --deploydir ${DEPLOYDIR} --targetusername UAT --testlevel ${TEST_LEVEL}"
		//    if (rc != 0) {
		//        error 'Salesforce deploy failed.'
		//    }
		//}
	    }
	}
}

def command(script) {
    if (isUnix()) {
        return sh(returnStatus: true, script: script);
    } else {
	return bat(returnStatus: true, script: script);
    }
}
