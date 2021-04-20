
def props = readProperties  file:'/projectProperties/${branchName}.properties'
def L_SH_BRANCH= props['SH_BRANCH']
env.SH_PARENTBRANCH= props['SH_PARENTBRANCH']
env.PROJECTNAME= props['PROJECTNAME']
env.SF_USERNAME= props['SF_USERNAME']
env.VERIFYONLY= props['VERIFYONLY']
env.TEST_LEVEL= props['TEST_LEVEL']
env.SF_INSTANCE_URL= props['SF_INSTANCE_URL']
env.RUNVERACODE= props['RUNVERACODE']
env.RUNSONAR= props['RUNSONAR']
env.RUNSONAR= props['UPLOADNEXUS']
if(${TRIGGERBRANCH} == null || ${TRIGGERBRANCH} == ""){
	env.SH_BRANCH=${L_SH_BRANCH}
}
else{
	env.SH_BRANCH=${TRIGGERBRANCH}
}

#!groovy
import groovy.lang.Binding
node { 
	withEnv(["HOME=${env.WORKSPACE}"]) {
		stage('Checkout') {
			cleanWs()
			
			rm=checkout([$class: 'GitSCM', branches: [[name: '**']], browser: [$class: 'BitbucketWeb', repoUrl: '${SC_URL}'], extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'metadata']], userRemoteConfigs: [[credentialsId: 'GITCREDS', url: '${SC_URL}']]])

            env.SH_BRANCH = "devtest"
            env.SH_PARENTBRANCH = "dev"

			sh(returnStatus: true, script: """#!/bin/bash
			mkdir "delta"
			mkdir "convertdelta"
			cd metadata
			git checkout ${env.SH_BRANCH}
			diffStr=`git diff --name-only origin/${env.SH_PARENTBRANCH} ${env.SH_BRANCH}`
			diffStr=`echo \$diffStr | sed 's/ /;/g'`
			echo \$diffStr
			IFS=';' read -r -a pathArray <<< "\$diffStr"
			echo "Size is \${#pathArray[@]}"
			for ele in "\${pathArray[@]}"
            do
            echo "\$ele"
            ele=`echo \$ele | sed 's/\\(\\[\\@\\]\\)//g'`
            echo "\$ele"
            lpath=""
            if [[ "\$ele" == *\\/* ]]
				then
					lpath=`echo \$ele | sed 's|\\(.*\\)/.*|\\1|'`
					lpath=`echo \$lpath | sed 's|force-app/main/default/||'`
					echo "Folder path is \$lpath"
					mkdir -p ../delta/\$lpath
                fi
			cp \$ele ../delta/\$lpath/
			cp "\$ele-meta.xml" ../delta/\$lpath/
			done
			cp manifest/package.xml ../delta/
			cd ../convertdelta
			sfdx force:project:create -n tempProject
			cd tempProject
			mkdir temp
			cp -R ../../delta/* ../../convertdelta/tempProject/temp
			sfdx force:mdapi:convert -r ./temp -d ./force-app
			sfdx force:source:convert --rootdir ./force-app --outputdir ./unmanaged
			rm -r ../../delta/*
			cp -R unmanaged ../../delta
			cd ../../delta
			zip -r unmanaged.zip unmanaged/*
			""")
		}
	}
}

node {

    def SF_CONSUMER_KEY = env.SF_CONSUMER_KEY
    def SF_USERNAME = env.SF_USERNAME
    def SERVER_KEY_CREDENTIALS_ID = env.SERVER_KEY_CREDENTIALS_ID
    def TEST_LEVEL = env.TEST_LEVEL
    def SF_INSTANCE_URL = env.SF_INSTANCE_URL ?: "https://login.salesforce.com"
    def SC_URL = env.SC_URL
	
    def DEPLOYDIR='src/deploy'
    def MAINSOURCEDIR='src/force-app/main/default/'
    def PACKAGESOURCEDIR='src/manifest/package.xml'
    
    def toolbelt = tool 'toolbelt'

    // -------------------------------------------------------------------------
    // Check out code from source control.
    // -------------------------------------------------------------------------

    stage('checkout source') {
        checkout scm
	    
	checkout changelog: false, poll: false, scm: [$class: 'GitSCM', branches: [[name: '*/master']], extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'src']], userRemoteConfigs: [[credentialsId: 'github', url: '${SC_URL}']]]
	    
	command "mkdir ${DEPLOYDIR}"
	command "cp ${PACKAGESOURCEDIR} ${DEPLOYDIR}/package.xml"
	command "cp -R ${MAINSOURCEDIR} ${DEPLOYDIR}"
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
		
		post { 
			always { 
				cleanWs()
			}
		}
		
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
