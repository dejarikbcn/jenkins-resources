timestamps{
	def GIT_PROJECT_PARAM = "1370-EJCTRA-RMA.git"
	def GIT_BRANCH_PARAM = params.git_branch
	node('am19') {
		stage('Setting up environment'){
		}
		stage ('Checking out GitBlit branch') {
			wrap([$class: 'BuildUser']) {
				configFileProvider([configFile(fileId: '8d41fb14-a333-4f6a-8d2d-fb02d35cac8f', variable: 'JENKINS_DEPLOYMENT_SETTINGS')]) {
					def GLOBAL_DEPLOYMENT_SETTINGS = readYaml(file:JENKINS_DEPLOYMENT_SETTINGS)
					def GITBLIT_URL = GLOBAL_DEPLOYMENT_SETTINGS.repository.url+"/"+GIT_PROJECT_PARAM
					sh "rm -f ${WORKSPACE}/credential-helper.sh"
					sh "echo '#!/bin/bash' >> ${WORKSPACE}/credential-helper.sh"
					sh "echo 'echo username=${BUILD_USER_ID}' >> ${WORKSPACE}/credential-helper.sh"
					sh "echo 'echo password=\044git_password' >> ${WORKSPACE}/credential-helper.sh"
					sh "chmod +x ${WORKSPACE}/credential-helper.sh"
					sh "rm -fdR ./gitcontents"
					sh "mkdir -p ./gitcontents"
					dir ('./gitcontents') {
						sh 'git init'
						sh "git config user.email \"${BUILD_USER_EMAIL}\""
						sh "git config user.name \"${BUILD_USER_ID}\""
						sh "git config credential.helper \"/bin/bash ${WORKSPACE}/credential-helper.sh\""
						sh "git remote add origin ${GITBLIT_URL}"
						sh "git fetch --tags origin 'refs/heads/${GIT_BRANCH_PARAM}:refs/remotes/origin/${GIT_BRANCH_PARAM}'"
						sh "git checkout ${GIT_BRANCH_PARAM}"
						sh "git reset --hard refs/remotes/origin/${GIT_BRANCH_PARAM}"
						sh 'git clean -fd'
					}
				}
			}
		}
		stage ('Building EAR with Maven') {
			dir ('./gitcontents') {
				def DIR_antLib = "/opt/jenkins-addons/lib"
				def DIR_weblogic = "/opt/jenkins-addons/wlserver"
				def DIR_antScripts = "/opt/jenkins-addons/ant-scripts"
				def ENTORN = "DES"
				sh "mvn -s /opt/jenkins-addons/user-settings.xml package -Doit.entorn=${ENTORN} -DDIR_antLib=${DIR_antLib} -DDIR_weblogic=${DIR_weblogic} -DDIR_antScripts=${DIR_antScripts}"
			}
		}
		stage ('Transferring EAR to target server') {
			dir ('./gitcontents') {
				configFileProvider([configFile(fileId: '8d41fb14-a333-4f6a-8d2d-fb02d35cac8f', variable: 'JENKINS_DEPLOYMENT_SETTINGS')]) {
					def GLOBAL_DEPLOYMENT_SETTINGS = readYaml(file:JENKINS_DEPLOYMENT_SETTINGS)
					def FTP_URL = GLOBAL_DEPLOYMENT_SETTINGS.ftp.url
					def FTP_PASSWORD = GLOBAL_DEPLOYMENT_SETTINGS.ftp.username
					def FTP_USERNAME = GLOBAL_DEPLOYMENT_SETTINGS.ftp.password
					def FTP_TARGET_FOLDER = GLOBAL_DEPLOYMENT_SETTINGS.ftp.targetFolder	
					wrap([$class: 'MaskPasswordsBuildWrapper', 
						varPasswordPairs: [
							[password: "${FTP_URL}", var: 'FTP_URL'],
							[password: "${FTP_USERNAME}", var: 'FTP_USERNAME'],
							[password: "${FTP_PASSWORD}", var: 'FTP_PASSWORD']
						]]) {
						def EAR_LOCATION = sh(returnStdout:true,script: 'find . -name "*dynamic.ear"').trim()
						def EAR_FILE = EAR_LOCATION.substring(EAR_LOCATION.lastIndexOf('/')+1,EAR_LOCATION.length())
						sh "sshpass -p ${FTP_PASSWORD} scp -o StrictHostKeyChecking=no ${EAR_LOCATION} ${FTP_USERNAME}@${FTP_URL}:${FTP_TARGET_FOLDER}/${EAR_FILE}"
						def EXPLODED = "S"
						if (EXPLODED == 'S'){
							stage('Exploding EAR'){
								def DEPLOYMENT_NAME = "RMAEAR"
								sh "sshpass -p ${FTP_PASSWORD} ssh -o StrictHostKeyChecking=no -t ${FTP_USERNAME}@${FTP_URL} 'rm -rf ${FTP_TARGET_FOLDER}/${DEPLOYMENT_NAME}';"
								sh "sshpass -p ${FTP_PASSWORD} ssh -o StrictHostKeyChecking=no -t ${FTP_USERNAME}@${FTP_URL} 'unzip ${FTP_TARGET_FOLDER}/${EAR_FILE} -d ${FTP_TARGET_FOLDER}/${DEPLOYMENT_NAME}';"
							}
						}
					}
				}
			}
		}
		stage ('Requesting Weblogic Deploy') {
			dir ('./gitcontents') {
				configFileProvider([configFile(fileId: '8d41fb14-a333-4f6a-8d2d-fb02d35cac8f', variable: 'JENKINS_DEPLOYMENT_SETTINGS')]) {
					//def MODULE_DEPLOYMENT_SETTINGS = readYaml(file:"sic/deployment.yaml")
					def GLOBAL_DEPLOYMENT_SETTINGS = readYaml(file:JENKINS_DEPLOYMENT_SETTINGS)
					//def DEPLOYMENT_NAME = MODULE_DEPLOYMENT_SETTINGS.deployment.moduleName
					//def DEPLOYMENT_DOMAIN = MODULE_DEPLOYMENT_SETTINGS.deployment.domain
					def DEPLOYMENT_NAME = 'RMAEAR'
					def DEPLOYMENT_DOMAIN = 'justicia_domain'
					def FTP_URL = GLOBAL_DEPLOYMENT_SETTINGS.ftp.url
					def FTP_PASSWORD = GLOBAL_DEPLOYMENT_SETTINGS.ftp.username
					def FTP_USERNAME = GLOBAL_DEPLOYMENT_SETTINGS.ftp.password
					def DEPLOYMENT_URL = GLOBAL_DEPLOYMENT_SETTINGS.appserver.domains["${DEPLOYMENT_DOMAIN}"].url
					def DEPLOYMENT_PORT = GLOBAL_DEPLOYMENT_SETTINGS.appserver.domains["${DEPLOYMENT_DOMAIN}"].port
					wrap([$class: 'MaskPasswordsBuildWrapper', 
						varPasswordPairs: [
							[password: "${DEPLOYMENT_URL}", var: 'DEPLOYMENT_URL'],
							[password: "${DEPLOYMENT_PORT}", var: 'DEPLOYMENT_PORT'],
							[password: "${FTP_URL}", var: 'FTP_URL'],
							[password: "${FTP_PASSWORD}", var: 'FTP_PASSWORD'],
							[password: "${FTP_USERNAME}", var: 'FTP_USERNAME']
						]]) {
						sh "sshpass -p ${FTP_PASSWORD} ssh -o StrictHostKeyChecking=no -t ${FTP_USERNAME}@${FTP_URL} 'sh /home/fwadmin/wls/CuaDeploys/deployer_jenkins.sh -adminurl t3://${DEPLOYMENT_URL}:${DEPLOYMENT_PORT} -name ${DEPLOYMENT_NAME}';"
					}
				}
			}
		}
		stage ('Cleaning out contents'){
			sh "rm -f ${WORKSPACE}/credential-helper.sh"
			sh "rm -fdR ./gitcontents"
		}
    }
}