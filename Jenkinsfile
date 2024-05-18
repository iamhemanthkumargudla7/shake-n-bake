import groovy.json.JsonSlurperClassic
import static java.util.Calendar.*
import groovy.time.TimeCategory

def static getMasterEnvironment(final script, final String application, final String version, final String url, final String token) {

  def envListStr = script.sh(returnStdout: true, script: "java -jar eod-cli.jar environment list --app='${application}' --version='${version}' --token=${token} --host=${url}").trim()
  script.echo "Environment List: ${envListStr}"
  def envList = new JsonSlurperClassic().parseText(envListStr)

  if (envList.size() > 1) {
    throw new IllegalStateException("There are already [${envList.size()}] environments that exist for [${application} - ${version}]")
  }

  return envList[0]
}

def static provisionEnvironment(final script, final String application, final String version, final String description, final String url, final String token) {
  def envObj = script.sh(returnStdout: true, script: "java -jar eod-cli.jar environment create --app='${application}' --version='${version}' --category='DevEnvironment' --reason='${description}' --token=${token} --host=${url}").trim()
  script.echo "Provisioned Environment: ${envObj}"
  def data = new JsonSlurperClassic().parseText(envObj)
  return data
}

def static resetMaster(final script, final String application, final String version, final String eodEnvironment, final String url, final String token) {
  script.sh """
    java -jar eod-current-process.jar reset-master --app='${application}' --version='${version}' --environment='${eodEnvironment}' --wait --token=${token}
  """
  return getMasterEnvironment(script, application, version, url, token)
}

def static getApplication(final script, final String application, final String url, final String token) {

  def appListStr = script.sh(returnStdout: true, script: "java -jar eod-cli.jar application inspect --app='${application}' --token=${token} --host=${url}").trim()
  script.echo "Application: ${appListStr}"
  def appList = new JsonSlurperClassic().parseText(appListStr)

  if (appList.size() == 0) {
    throw new IllegalStateException("Application [${application}] does not exist.")
  }

  return appList[0]
}

def static getLatestVersion(final script, final String application, final String url, final String token) {
  def app = getApplication(script, application, url, token)

  def conn = new URL(url + "/applications/${app.id}/versions/latest").openConnection();
  conn.setRequestProperty("Authorization", "Bearer ${token}")
  def response = conn.getResponseCode();

  if (!200.equals(response)) {
    throw new IllegalStateException("Unable to retrieve latest version for application [${app.name}][code=${response}].")
  }

  def results = new JsonSlurperClassic().parseText("${conn.getInputStream().getText()}")
  return results.version;
}

def static getVersion(final script, final String application, final String version, final String url, final String token) {
  def app = getApplication(script, application, url, token)

  def conn = new URL(url + "/applications/${app.id}/versions?name=${version}").openConnection();
  conn.setRequestProperty("Authorization", "Bearer ${token}")
  def response = conn.getResponseCode();

  if (!200.equals(response)) {
    throw new IllegalStateException("Unable to retrieve version [${version}] for application [${app.name}][code=${response}].")
  }

  def results = new JsonSlurperClassic().parseText("${conn.getInputStream().getText()}")
  return results.versions[0]?.version;
}

pipeline {
    agent {
      node {
        label 'eod-process'
      }
    }

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['STAGING', 'DEVELOPMENT', 'PRODUCTION'], description: 'Select which environment to execute against')
        choice(name: 'MASTER_TYPE', choices: ['EXISTING', 'CREATED'], description: 'Select whether the master already exists or not.')
        string(name: 'APPLICATION', description: 'The application name',defaultValue: 'VMWare_Shake-n-bake')
        string(name: 'BASE_VERSION', description: 'The base version to shake off of', defaultValue: 'Base')
        string(name: 'PREVIOUS_VERSION', description: 'The previous version to retire',defaultValue: '-')
        string(name: 'NEW_VERSION', description: 'The name of the new version to create from BASE_VERSION',defaultValue: 'new-vesion')
        booleanParam(name: 'PROMPT_ENV_MODIFICATIONS', defaultValue: true, description: 'Prompt to verify environment modifications.')
        booleanParam(name: 'PROMPT_ENV_VERIFICATION', defaultValue: false, description: 'Prompt to verify environment validity.')
        booleanParam(name: 'CREATE_MEDIC_SNAPSHOTS', defaultValue: false, description: 'Create medic snapshots?')
        choice(name: 'PREVIOUS_VERSION_OPERATION', choices: ['IGNORE', 'DEACTIVATE', 'REFILL', 'FIXED-CHECKOUT', 'DESTROY'], description: 'What operation to perform on the previous version.')
        string(name: "POOLING_CONCURRENCY", description: "The number of concurrent pooling environments provisiong at once using the REFILL policy.", defaultValue: '25')
        string(name: "POOLING_INITIAL_SIZE", description: "The number of environments to initially set in the pool using the REFILL policy.", defaultValue: '0')
        string(name: 'DESTROY_LEAD_TIME', description: 'The amount of time in minutes after user notifications to destroy previous environments', defaultValue: '0')
	      string(name: 'RESOURCE_LOCK_NAME', description: 'Lock master provisioning based upon name, using - will default to application + version name', defaultValue: '-')
        choice(name: 'POOLING_POLICY', description: 'How to manage the pooling of environments outside of the REFILL policy.', choices: ['NONE', 'NEW-ONLY', 'NEW-AND-DEPRECATE'])
        string(name: 'POOL_SIZE', description: 'The number of environments desired to be maintained in the application version pool', defaultValue: '5')
        string(name: 'MAX_VERSION_POLICY_SIZE', description: 'The maximum number environments allowed for this new application version. Set as 0 to ignore.', defaultValue: '0')
        string(name: 'FIXED_CONFIG_NAME', description: 'The name of the yaml configuration to use for the FIXED-CHECKOUT configuration', defaultValue: '-')
        text(name: 'PRIMARY_MASTER_CONFIG', description: 'Configuration in yaml of the primary master', defaultValue: '')
    }

    stages {
        // Kicks off the current build process
        stage ('Initializing') {
          steps {
            echo "Initializing parameters for environment ${params.ENVIRONMENT}"

            script {
              int lead = Integer.parseInt("${params.DESTROY_LEAD_TIME}")
              if (lead < 0) {
                throw new IllegalStateException("DESTROY_LEAD_TIME [${params.DESTROY_LEAD_TIME}] must be positive.")
              }

              int concurrent = Integer.parseInt("${params.POOLING_CONCURRENCY}")
              if (concurrent < 1) {
                throw new IllegalStateException("POOLING_CONCURRENCY [${params.POOLING_CONCURRENCY}] must be greater than 0.")
              }

              int poolInitSize = Integer.parseInt("${params.POOLING_INITIAL_SIZE}")
              if (poolInitSize < 0) {
                throw new IllegalStateException("POOLING_INITIAL_SIZE [${params.POOLING_INITIAL_SIZE}] must be positive.")
              }

              int poolSize = Integer.parseInt("${params.POOL_SIZE}")
              if (poolSize < 0) {
                throw new IllegalStateException("POOL_SIZE [${params.POOL_SIZE}] must be positive.")
              }

              masterEnvFormatted = ""
              validationEnvUUID = ""
	            lockName = ""

              if ("${params.ENVIRONMENT}" == "DEVELOPMENT") {
                echo 'Setting parameters for DEVELOPMENT'
                env.EOD_SERVICE_URL = "http://ipdevenvdev01.ip.devcerner.net:8085/eod-service"
                env.EOD_TOKEN = "5qWZl3V3F3053dfgPYXGmcIvOWAHcso5vCfW58kTwLtTM2cQ7OJQm3AtytTuTD0hbuaDTA6QJIxQx4vcG3bqTto886iheCT8BdJjdTmOUu7rKgzsdteGJSsgaLJX8nOL6Z"
                env.RD_URL = "http://dev.ipdevenv.ip.devcerner.net:4440"
                env.RD_USER = "eodadmin"
                env.RD_PASSWORD = "FlyingMonkeyHiddenPanda321!"
                env.VRA_HOST = "vradev.northamerica.cerner.net"
		            env.RD_AGENT_RESET_ID = "c229ff83-4df5-4455-9a39-465f0d4db078"
                env.TEAMS_CHANNEL = "https://outlook.office.com/webhook/5cff0d1c-8f72-41b5-81bf-2235195f1164@fbc493a8-0d24-4454-a815-f4ca58e8c09d/IncomingWebhook/29f1f5106dea4ae48dea98b9e8624f0e/548b6cd9-0708-43db-89c4-bc6a4b95b271"
                env.TEAMS_FAILURE_CHANNEL = "${env.TEAMS_CHANNEL}"
                env.POOLING_STORY_ID = ""
                env.POOLING_TOKEN = ""
                env.CHRONICLE_STORY_ID = ""
                env.CHRONICLE_STORY_SECRET = ""
              } else if ("${params.ENVIRONMENT}" == "STAGING") {
                echo 'Setting parameters for STAGING'
                env.EOD_SERVICE_URL = "http://staging.ipdevenv.ip.devcerner.net/eod-service"
                env.EOD_TOKEN = "Po7F6pn3FzxPtl60Atqb9lV19ngfgPddo3F5m661dAMqP3RFXi6ZvwHu8paiixsO2Umzy6DVp8kikv1B0yZfuoIi5ORisq2rV1s6kFhk9kUtexNNjmWGrak4dhGvtu92F5"
                env.RD_URL = "http://staging.ipdevenv.ip.devcerner.net:4440"
                env.RD_USER = "eodadmin"
                env.RD_PASSWORD = "FlyingMonkeyHiddenPanda321!"
                env.VRA_HOST = "vracrt.northamerica.cerner.net"
		            env.RD_AGENT_RESET_ID = "c229ff83-4df5-4455-9a39-465f0d4db078"
                env.TEAMS_CHANNEL = "https://outlook.office.com/webhook/5cff0d1c-8f72-41b5-81bf-2235195f1164@fbc493a8-0d24-4454-a815-f4ca58e8c09d/IncomingWebhook/29f1f5106dea4ae48dea98b9e8624f0e/548b6cd9-0708-43db-89c4-bc6a4b95b271"
                env.TEAMS_FAILURE_CHANNEL = "${env.TEAMS_CHANNEL}"
                env.POOLING_STORY_ID = "2fe26f3e-32c3-4e05-8027-c8513dce77c6"
                env.POOLING_TOKEN = "oP6zNcmz6xskdDm8pkQbK2bqih0g3TOkGK1lXIIU43PAKdUnvhFUF4gnD8Yrmj8FOYRsM9U2MgmXcqOWxSmN16D0DL1UNBRGFmNXpywHZPad76hhErziQMsFR5CSIXXgmr"
                env.CHRONICLE_STORY_ID = "0af48177-4ab5-4995-b892-8f5e36826d2d"
                env.CHRONICLE_STORY_SECRET = "7446785f-163a-45aa-b29b-93381925180c"
              } else if ("${params.ENVIRONMENT}" == "PRODUCTION") {
                echo 'Setting parameters for PRODUCTION'
                env.EOD_SERVICE_URL = "http://ipdevenv.ip.devcerner.net/eod-service"
                env.EOD_TOKEN = "Po7F6pn3FzxPtl60Atqb9lV19ngfgPddo3F5m661dAMqP3RFXi6ZvwHu8paiixsO2Umzy6DVp8kikv1B0yZfuoIi5ORisq2rV1s6kFhk9kUtexNNjmWGrak4dhGvtu92F5"
                env.RD_URL = "http://ipdevenv.ip.devcerner.net:4440"
                env.RD_USER = "eodadmin"
                env.RD_PASSWORD = "FlyingMonkeyHiddenPanda321!"
                env.VRA_HOST = "vraprd.northamerica.cerner.net"
		            env.RD_AGENT_RESET_ID = "c229ff83-4df5-4455-9a39-465f0d4db078"
                env.TEAMS_CHANNEL = "https://cernerprod.webhook.office.com/webhookb2/ccecd4ae-bc04-4a63-b962-c8c6186c66a7@fbc493a8-0d24-4454-a815-f4ca58e8c09d/IncomingWebhook/de62a82859a2442fa4126f4eb823f487/548b6cd9-0708-43db-89c4-bc6a4b95b271"
                env.TEAMS_FAILURE_CHANNEL = "https://cernerprod.webhook.office.com/webhookb2/ccecd4ae-bc04-4a63-b962-c8c6186c66a7@fbc493a8-0d24-4454-a815-f4ca58e8c09d/IncomingWebhook/b4e9df1b0ab6492485b233111e094808/548b6cd9-0708-43db-89c4-bc6a4b95b271"
                env.POOLING_STORY_ID = "1a3553e8-4b49-4977-8b4d-44a48c9e93ed"
                env.POOLING_TOKEN = "PV1x9vkBmoymyU3dTPzNrYkM8pkReuXvDn3lhs6joUM9KBXuNKi9BXibMsk5yKu4phKNTLVTzuFjtJzQj0eO9Ai4SSxYMaLPdRTBgPiLTapMIu2E4Y2E6D3zY6Sgp0iIV2"
                env.CHRONICLE_STORY_ID = "c7d3722b-f02b-431c-b1a7-628fe41e13ef"
                env.CHRONICLE_STORY_SECRET = "b68d99bc-4d37-47bc-9696-3898311e9ff6"
              } else {
                currentBuild.result = 'ABORTED'
                error("UNKNOWN ENVIRONMENT: ${params.ENVIRONMENT}")
              }

      	      if ("${params.RESOURCE_LOCK_NAME}" == "-") {
      		      env.LOCK_NAME = "${params.APPLICATION}"
      	      } else {
                env.LOCK_NAME = "${params.RESOURCE_LOCK_NAME}"
              }
            }

            buildName "#${BUILD_NUMBER}-${params.NEW_VERSION}"

            echo "EOD URL:       ${env.EOD_CLIENT_URL}"
            echo "EOD PORT:      ${env.EOD_CLIENT_PORT}"
            echo "Rundeck URL:   ${env.RD_URL}"
            echo "vRealize URL:  ${env.VRA_HOST}"
            echo "Teams Channel: ${env.TEAMS_CHANNEL}"
	          echo "Lock Name:     ${env.LOCK_NAME}"

          }
        }
        stage ('Installing Tools') {
            steps {
                artifactResolver artifacts: [
                  artifact(
                    groupId: 'com.cerner.abilities',
                    artifactId: 'eod-cli',
                    classifier: 'jar-with-dependencies',
                    extension: 'jar',
                    version: '1.3',
                    targetFileName: './eod-cli.jar'
                  ),
                  artifact(
                    groupId: 'com.cerner.abilitieslab',
                    artifactId: 'eod-current-process',
                    classifier: 'jar-with-dependencies',
                    extension: 'jar',
                    version: '1.43',
                    targetFileName: './eod-current-process.jar'
                  ),
                  artifact(
                    groupId: 'org.rundeck',
                    artifactId: 'rundeck-cli-all',
                    extension: 'jar',
                    version: '1.1.0',
                    targetFileName: './rundeck-cli.jar'
                  )
                ],
                targetDirectory: WORKSPACE
            }
        }

        stage ('Resolve Input') {
          steps {
            sh """
              java -jar eod-cli.jar application inspect --app='${params.APPLICATION}' --token=${env.EOD_TOKEN} --host=${EOD_SERVICE_URL}
            """
            sh """
              java -jar eod-cli.jar version inspect --app='${params.APPLICATION}' --version='${params.BASE_VERSION}' --token=${env.EOD_TOKEN} --host=${EOD_SERVICE_URL}
            """
            script {

              if ("${params.PREVIOUS_VERSION}".equalsIgnoreCase("latest")) {
                env.PREVIOUS_EOD_VERSION = getLatestVersion(this,"${params.APPLICATION}", "${EOD_SERVICE_URL}","${env.EOD_TOKEN}").name
              } else {
                env.PREVIOUS_EOD_VERSION = "${params.PREVIOUS_VERSION}"
              }

              echo "Previous Version:       ${env.PREVIOUS_EOD_VERSION}"

              if ("${env.PREVIOUS_EOD_VERSION}" == "${params.NEW_VERSION}") {
                throw new IllegalStateException("The previous version matches the new version [${params.NEW_VERSION}].")
              }
            }
          }
        }

        stage ('Environment Modifications - Existing Master') {
          steps {
            script {
              if("${params.MASTER_TYPE}" == 'EXISTING') {
                if ("${params.PROMPT_ENV_MODIFICATIONS}" == "true") {
                  office365ConnectorSend message: "**_[${params.APPLICATION}] - ${params.NEW_VERSION}:_**  Shake and bake process has been started and is awaiting confirmation that modifications to master environment were performed.\n\n\n\nPreparing to ${params.PREVIOUS_VERSION_OPERATION} previous version ${env.PREVIOUS_EOD_VERSION}.", status: "Awaiting master environment modifications.", webhookUrl: "${env.TEAMS_CHANNEL}", color: "ffe070"

                  def response = input(submitterParameter: 'submitter', message: "[${params.APPLICATION}] - ${params.NEW_VERSION} Have master environment changes been completed?\n\nPreparing to ${params.PREVIOUS_VERSION_OPERATION} previous version ${env.PREVIOUS_EOD_VERSION}.")
                  office365ConnectorSend message: "**_[${params.APPLICATION}] - ${params.NEW_VERSION}:_**  Environment modifications were validated by ${response}.", status: "Preparing environment for shake and bake.", webhookUrl: "${env.TEAMS_CHANNEL}", color: "007cc3"
                } else {
                  office365ConnectorSend message: "**_[${params.APPLICATION}] - ${params.NEW_VERSION}:_**  Shake and bake process has been started.\n\n\n\nPreparing to ${params.PREVIOUS_VERSION_OPERATION} previous version ${env.PREVIOUS_EOD_VERSION}.", status: "Preparing environment for shake and bake.", webhookUrl: "${env.TEAMS_CHANNEL}", color: "007cc3"
                }
              }
            }
          }
        }

        stage ('Shake') {
            steps {
              sh """
                java -jar eod-cli.jar version update --app='${params.APPLICATION}' --version='${params.BASE_VERSION}' --status=STABLE --token=${env.EOD_TOKEN} --host=${EOD_SERVICE_URL}
                java -jar eod-cli.jar version shake --source-app='${params.APPLICATION}' --source-name='${params.BASE_VERSION}' --target-app='${params.APPLICATION}' --target-name='${params.NEW_VERSION}' --token=${env.EOD_TOKEN} --host=${EOD_SERVICE_URL}
              """
            }
        }

        stage ('Provision Master') {
          steps {
            lock (resource: "shake-n-bake-${env.LOCK_NAME}") {
              script {
                def master = getMasterEnvironment(this, "${params.APPLICATION}","${params.NEW_VERSION}","${EOD_SERVICE_URL}","${env.EOD_TOKEN}")
                /*
                 * Master has not been created yet, schedule a provision attempt as normal.  If it fails, try to reset the master at least once
                 */
                if (master == null) {
                  try {
                      master = provisionEnvironment(this, "${params.APPLICATION}","${params.NEW_VERSION}","Automated Jenkins Build: ${env.BUILD_ID} Validation", "${EOD_SERVICE_URL}","${env.EOD_TOKEN}")
                  } catch (Exception e) {
                      master = resetMaster(this, "${params.APPLICATION}","${params.NEW_VERSION}","${params.ENVIRONMENT}", "${EOD_SERVICE_URL}","${env.EOD_TOKEN}")
                  }
                } else if ("${master?.status}" == 'FAILED') {
                  master = resetMaster(this, "${params.APPLICATION}","${params.NEW_VERSION}","${params.ENVIRONMENT}", "${EOD_SERVICE_URL}","${env.EOD_TOKEN}")
                } else if ("${master?.name}".startsWith('mstr-') && "${master?.status}" == 'AVAILABLE') {
                  echo "Existing master [${master?.name}] is available. Moving on..."
                } else {
                  throw new IllegalStateException("Existing master [${master?.name}] is not in a viable state [${master?.status}].")
                }

                def format = "%-17s %20s %22s\n"
                builder = new StringBuilder()
                builder.append('Environment ID:   ' + master.id + '\n')
                builder.append('Environment Name: ' + master.name + '\n')
                builder.append('\n')
                builder.append(String.format(format, 'Name', 'IP Address', 'Role'))
                builder.append(String.format(format, '-----------------', '--------------------','----------------------'))
                master.devices.each { device ->
                  echo "${device}"
                  builder.append(String.format(format, "${device.name}", "${device.ip_address}", "${device.role}"))
                }

                masterEnvFormatted = builder

              }
            }
          }
        }

        stage ('Environment Modifications - Created Master') {
          steps {
            script {
              if("${params.MASTER_TYPE}" == 'CREATED') {
                office365ConnectorSend message: "**_[${params.APPLICATION}] - ${params.NEW_VERSION}:_**  Shake and bake process has successfully created a master environment and is awaiting modifications.\n\n<pre>${masterEnvFormatted}</pre>", status: "Awaiting master environment modifications.", webhookUrl: "${env.TEAMS_CHANNEL}", color: "ffe070"

                def response = input(submitterParameter: 'submitter', message: "[${params.APPLICATION}] - ${params.NEW_VERSION} Have master environment changes been completed?\n\nPreparing to ${params.PREVIOUS_VERSION_OPERATION} previous version ${env.PREVIOUS_EOD_VERSION}.")
                office365ConnectorSend message: "**_[${params.APPLICATION}] - ${params.NEW_VERSION}:_**  Environment modifications were validated by ${response}.", status: "Preparing environment for shake and bake.", webhookUrl: "${env.TEAMS_CHANNEL}", color: "007cc3"
              }
            }
          }
        }

      stage ('Medic Snapshot') {
        steps {
          script {
            try {
              if ("${params.CREATE_MEDIC_SNAPSHOTS}" == "true" && "${params.PRIMARY_MASTER_CONFIG}" != "") {
                def yaml = readYaml text: "${params.PRIMARY_MASTER_CONFIG}"
                def config = yaml.config

                def domain = config.domain
                def appNode = config.masters.find { it.role == 'millennium-app'}
                def dbNode = config.masters.find { it.role == 'millennium-db'}

                echo "App Node: ${appNode}"
                echo "DB Node:  ${dbNode}"

                if (appNode != null && dbNode != null) {
                  //Retrieve the version id
                  def version = getVersion(this, "${params.APPLICATION}","${params.NEW_VERSION}","${EOD_SERVICE_URL}","${env.EOD_TOKEN}")
                  build job: 'medic/mill-snapshot', propagate: false, wait: false, parameters: [
                     [$class: 'StringParameterValue', name: 'APPLICATION',          value: "${params.APPLICATION}"],
                     [$class: 'StringParameterValue', name: 'VERSION',              value: "${params.NEW_VERSION}"],
                     [$class: 'StringParameterValue', name: 'VERSION_ID',           value: "${version?.uuid}"],
                     [$class: 'StringParameterValue', name: 'MILLENNIUM_DOMAIN',    value: "${config.domain}"],
                     [$class: 'StringParameterValue', name: 'MILLENNIUM_USER',      value: "${config.millennium_username}"],
                     [$class: 'StringParameterValue', name: 'MILLENNIUM_PASSWORD',  value: "${config.millennium_password}"],
                     [$class: 'StringParameterValue', name: 'MILLENNIUM_APP_NODE',  value: "${appNode.ip}"],
                     [$class: 'StringParameterValue', name: 'MILLENNIUM_DB_NODE',   value: "${dbNode.ip}"]
                   ]
                }
              }
            } catch (Exception e) {
              echo 'Failed to kick off snapshot: ' + e.toString()
            }
          }
        }
      }


        stage ('Bake') {
            steps {
              script {

                try {
                  sh """
                    java -jar eod-cli.jar version bake --app='${params.APPLICATION}' --version='${params.NEW_VERSION}' --token=${env.EOD_TOKEN} --host=${EOD_SERVICE_URL}
                  """
                } catch (Exception e) {
                    echo 'Exception occurred baking the master: ' + e.toString()
                    echo 'Attempting to reset status and bake again...'
                    sh """
                      java -jar eod-current-process.jar override-status --app='${params.APPLICATION}' --version='${params.NEW_VERSION}' --environment='${params.ENVIRONMENT}' --status='READY'
                    """
                    sh """
                      java -jar eod-cli.jar version bake --app='${params.APPLICATION}' --version='${params.NEW_VERSION}' --token=${env.EOD_TOKEN} --host=${EOD_SERVICE_URL}
                    """
                }

                //Verify the version status is in VALIDATING before moving forward
                def versionListStr = sh(returnStdout: true, script: "java -jar eod-cli.jar version inspect --app='${params.APPLICATION}' --version='${params.NEW_VERSION}' --token=${env.EOD_TOKEN} --host=${EOD_SERVICE_URL}").trim()
                echo "${versionListStr}"
                def versionList = new JsonSlurperClassic().parseText(versionListStr)
                if ("${versionList[0]?.status}" != 'VALIDATING') {
                  throw new IllegalStateException("Version is not in validating for [${params.APPLICATION} - ${params.NEW_VERSION}]. Actual = ${versionList[0]?.status}")
                }
              }
            }
        }

        stage ('Validation') {
            steps {
              script {
                def data = null
                def count = 1

                while (data == null && count < 4) {
                  try {
                    data = provisionEnvironment(this, "${params.APPLICATION}","${params.NEW_VERSION}","Automated Jenkins Build: ${env.BUILD_ID} Validation", "${EOD_SERVICE_URL}","${env.EOD_TOKEN}")

                    fatClient = "-"
                    validationEnvUUID = data.id

                    def format = "%-17s %20s %22s\n"
                    builder = new StringBuilder()
                    builder.append(String.format(format, 'Name', 'IP Address', 'Role'))
                    builder.append(String.format(format, '-----------------', '--------------------','----------------------'))
                    data.devices.each { device ->
                      echo "${device}"
                      builder.append(String.format(format, "${device.name}", "${device.ip_address}", "${device.role}"))
                      if ("${device.role}" == "FatClient") {
                        fatClient = device.host
                      }
                    }

                    if ("${params.PROMPT_ENV_VERIFICATION}" == "true") {
                      office365ConnectorSend message: "**_[${params.APPLICATION}] - ${params.NEW_VERSION}:_**  Test environment [${data.name}] successfully provisioned and requires validation.\n\n<pre>${builder}</pre>", status: "Awaiting Environment Validation", webhookUrl: "${env.TEAMS_CHANNEL}", color: "ffe070"
                      def response = input(submitterParameter: 'submitter', message: "[${params.APPLICATION}] - ${params.NEW_VERSION} Does the test environment appear healthy?")
                      office365ConnectorSend message: "**_[${params.APPLICATION}] - ${params.NEW_VERSION}:_**  Test environment was validated by ${response}.", status: "Marking new version as available", webhookUrl: "${env.TEAMS_CHANNEL}", color: "007cc3"
                    } else {
                      office365ConnectorSend message: "**_[${params.APPLICATION}] - ${params.NEW_VERSION}:_**  Test environment [${data.name}] successfully provisioned.\n\n<pre>${builder}</pre>", status: "Environment Validated Automatically", webhookUrl: "${env.TEAMS_CHANNEL}", color: "ffe070"
                    }
                  } catch (Exception e) {
                      echo "Exception occurred provisioning validation environment. Attempt ${count}: " + e.toString()
                      count++
                  }
                }

                if (data == null) {
                  throw new IllegalStateException("Unable to validate application version [${params.APPLICATION} - ${params.NEW_VERSION}].")
                }
              }
            }
        }

       stage ('Make Available') {
            steps {
              sh """
                java -jar eod-cli.jar version update --app='${params.APPLICATION}' --version='${params.NEW_VERSION}' --status=STABLE --token=${env.EOD_TOKEN} --host=${EOD_SERVICE_URL}
              """
              office365ConnectorSend message: "**_[${params.APPLICATION}] - ${params.NEW_VERSION}:_**  Marked version as stable.", status: "Preparing to ${params.PREVIOUS_VERSION_OPERATION} previous version ${env.PREVIOUS_EOD_VERSION}.", webhookUrl: "${env.TEAMS_CHANNEL}", color: "007cc3"
            }
       }

       stage ('Prepare Previous Version') {
            steps {
              script {
                expireDate = null
                expireDateStr = 'Not Applicable'

                if ("${env.PREVIOUS_EOD_VERSION}" != "-" && "${PREVIOUS_VERSION_OPERATION}" != "IGNORE") {
                  sh """
                    java -jar eod-current-process.jar override-status --app='${params.APPLICATION}' --version='${env.PREVIOUS_EOD_VERSION}' --environment='${params.ENVIRONMENT}' --status='RETIRED'
                  """

                  if ("${PREVIOUS_VERSION_OPERATION}" == "DESTROY" || "${PREVIOUS_VERSION_OPERATION}" == "REFILL" || "${PREVIOUS_VERSION_OPERATION}" == "FIXED-CHECKOUT") {

                    def cal = Calendar.instance//it return same time as new Date()
                    cal.setTimeZone(TimeZone.getTimeZone("America/Chicago"));
                    expireDate = new Date(cal.getTimeInMillis() + (Integer.parseInt("${DESTROY_LEAD_TIME}") * 60000))

                    echo "${expireDate}"
                    expireDateStr = expireDate.format("MM/dd/yyyy HH:mm")
                    sh """
                      java -jar eod-current-process.jar expire-all --app='${params.APPLICATION}' --version='${env.PREVIOUS_EOD_VERSION}' --environment='${params.ENVIRONMENT}' --date='${expireDateStr}'
                    """
                  }
                }
              }
            }
       }

       stage ('Announcements') {
            steps {
              script {
                def previousEmailMsg = ''
                def previousYammerMsg = ''
                if ("${env.PREVIOUS_EOD_VERSION}" != "-" && ("${PREVIOUS_VERSION_OPERATION}" == "DESTROY" || "${PREVIOUS_VERSION_OPERATION}" == "REFILL" || "${PREVIOUS_VERSION_OPERATION}" == "FIXED-CHECKOUT")) {
                  previousYammerMsg = "\n\nThe ${env.PREVIOUS_EOD_VERSION} environments will be destroyed at ${expireDateStr}."
                  previousEmailMsg = "<br><br>The previous version ${env.PREVIOUS_EOD_VERSION} will start being destroyed at ${expireDateStr}."
                }

                emailext (
                    subject: "Environments On Demand : ${params.ENVIRONMENT} - New Application Announcement",
                    body: """[Application Version Available: ${params.APPLICATION}] The ${params.NEW_VERSION} application version is now available.${previousEmailMsg}""",
                    mimeType: 'text/html',
                    to: "deven.hammerschmidt@cerner.com",
                    from: "no-reply@cerner.com"
                )

                yammerMessage = "<pre>[Application Version Available: ${params.APPLICATION}] The ${params.NEW_VERSION} application version is now available.${previousYammerMsg}"
                yammerMessage = yammerMessage + "</pre>"

                office365ConnectorSend message: "**_[${params.APPLICATION}] - ${params.NEW_VERSION}:_**  Notification post to yammer was made.  Yammer message:\n\n\n\n ${yammerMessage}", status: "Yammer update email sent", webhookUrl: "${env.TEAMS_CHANNEL}", color: "ffe070"
              }
            }
       }

       stage ('Destroy and Refill') {
            steps {
              script {
                echo "Destroy and Refill: ${expireDate}"
                if (expireDate) {
                  def now = new Date()
                  def time = expireDate.getTime() - now.getTime()
                  env.exit = false
                  echo "Destroy and Refill: ${time}"

                  if (time > 0) {
                    try {
                      timeout(time: "$time", unit: 'MILLISECONDS'){
                        def result = input id: 'Continue', message: 'Wait for entire time?', ok: 'Confirm', parameters: [choice(choices: ['Continue Now','Cancel'], description: '', name: 'CONTINUATION_OPTION')]
                        echo "Destroy and Refill: ${result}"
                        if ("Cancel".equals(result)) {
                          env.exit = true
                        }
                      }
                    } catch(err) { } //Timeout occurred

                  }
                }

                echo "${env.exit}"

                if ("${env.exit}" == "false") {
                  if ("${PREVIOUS_VERSION_OPERATION}" == "DESTROY") {
                    office365ConnectorSend message: "**_[${params.APPLICATION}] - ${params.NEW_VERSION}:_**  Starting destroys on ${env.PREVIOUS_EOD_VERSION}.", status: "Destroying previuos environments.", webhookUrl: "${env.TEAMS_CHANNEL}", color: "007cc3"
                    sh """
                      java -jar eod-current-process.jar destroy-all --app='${params.APPLICATION}' --version='${env.PREVIOUS_EOD_VERSION}' --environment='${params.ENVIRONMENT}' --token='${env.EOD_TOKEN}'
                    """
                  } else if("${PREVIOUS_VERSION_OPERATION}" == "REFILL") {
                    if ("${params.POOLING_INITIAL_SIZE}" != "0") {
                      office365ConnectorSend message: "**_[${params.APPLICATION}] - ${params.NEW_VERSION}:_**  Starting to destroy and refill pool with ${params.POOLING_INITIAL_SIZE} environments.", status: "Destroying ${env.PREVIOUS_EOD_VERSION} and filling ${params.NEW_VERSION}.", webhookUrl: "${env.TEAMS_CHANNEL}", color: "007cc3"
                      sh """
                        java -jar eod-current-process.jar reload-version --app='${params.APPLICATION}' --version='${params.NEW_VERSION}' --previous='${env.PREVIOUS_EOD_VERSION}' --environment='${params.ENVIRONMENT}' --storyId='${env.POOLING_STORY_ID}' --token='${env.POOLING_TOKEN}' --concurrent=${params.POOLING_CONCURRENCY} --count=${params.POOLING_INITIAL_SIZE} --maxResourcePercent=92
                      """
                    }
                  } else if("${PREVIOUS_VERSION_OPERATION}" == "FIXED-CHECKOUT") {
                    build job: 'eod-fixed-deployment', propagate: true, wait: true, parameters: [
                      [$class: 'StringParameterValue', name: 'ENVIRONMENT', value: "${params.ENVIRONMENT}"],
                      [$class: 'StringParameterValue', name: 'APPLICATION', value: "${params.APPLICATION}"],
                      [$class: 'StringParameterValue', name: 'VERSION', value: "${params.NEW_VERSION}"],
                      [$class: 'StringParameterValue', name: 'PREVIOUS_VERSION', value: "${env.PREVIOUS_EOD_VERSION}"],
                      [$class: 'StringParameterValue', name: 'CONFIG_FILE', value: "${params.FIXED_CONFIG_NAME}"]
                    ]
                  }
                }
              }
            }
       }

       stage ('Configure Policies') {
            steps {
              script {
                if ("${POOLING_POLICY}" == "NONE") {
                  echo "Ignoring pooling information"
                } else if ("${POOLING_POLICY}" == "NEW-ONLY") {
                  sh """
                    java -jar eod-current-process.jar set-pool-policy --app='${params.APPLICATION}' --version='${params.NEW_VERSION}' --environment="${params.ENVIRONMENT}" --token='${env.EOD_TOKEN}' --pool-size=${env.POOL_SIZE} --storyId='${env.CHRONICLE_STORY_ID}' --storySecret='${env.CHRONICLE_STORY_SECRET}'
                  """
                } else if ("${POOLING_POLICY}" == "NEW-AND-DEPRECATE") {
                  sh """
                    java -jar eod-current-process.jar set-pool-policy --app='${params.APPLICATION}' --version='${params.NEW_VERSION}' --environment="${params.ENVIRONMENT}" --token='${env.EOD_TOKEN}' --pool-size=${env.POOL_SIZE} --storyId='${env.CHRONICLE_STORY_ID}' --storySecret='${env.CHRONICLE_STORY_SECRET}' --reset-previous
                  """
                } else {
                  throw new IllegalStateException("Pooling policy [${POOLING_POLICY}] not known.")
                }

                if ("${params.MAX_VERSION_POLICY_SIZE}" != "0") {
                  sh """
                    java -jar eod-current-process.jar set-max-version-policy --app='${params.APPLICATION}' --version='${params.NEW_VERSION}' --environment="${params.ENVIRONMENT}" --token='${env.EOD_TOKEN}' --max-size=${params.MAX_VERSION_POLICY_SIZE} --storyId='${env.CHRONICLE_STORY_ID}' --storySecret='${env.CHRONICLE_STORY_SECRET}'
                  """
                }
              }
            }
       }

    }
    post {
      always {
        script {
          if ("${validationEnvUUID}" != "") {
            sh "java -jar eod-cli.jar environment destroy --ids=${validationEnvUUID} --token=${env.EOD_TOKEN} --host=${EOD_SERVICE_URL}"
          }
        }
      }
      failure {
        office365ConnectorSend message: "**_[${params.APPLICATION}] - ${params.NEW_VERSION}:_**  Shake and bake process failed and requires attention.", status: "Shake and bake process failed.", webhookUrl: "${env.TEAMS_FAILURE_CHANNEL}", color: "ff0000"
      }
      success {
        office365ConnectorSend message: "**_[${params.APPLICATION}] - ${params.NEW_VERSION}:_**  Shake and bake process completed successfully.", status: "Version completed.", webhookUrl: "${env.TEAMS_CHANNEL}", color: "78c346"
      }
    }
}
