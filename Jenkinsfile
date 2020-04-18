// This Jenkins pipeline script pulls New Relic Synthetics scripts from a GitHub repository and imports them into NR Synthetics.
// It requires the Pipeline Utility Steps and HTTP Request plugins.  You also need to add the New Relic API and Insights Insert Key
// into your Jenkins credential store.
pipeline {
    agent { docker { image 'node:6.3' } }
    stages {
        stage('build') {
            steps {
                println("Build step - nothing to see here now")
            }
        }
        stage('test') {
            steps {
                sh 'node -c *.js'
            }
        }
        stage('deploy') {
            environment {
              //Creds are defined in Jenkins to keep them secret.  You should give them your own meaningful name and change the ones I used below.
              nrApiKey = credentials('Shredtacular-Industries-Anewhouse-Admin-API-Key')
              nrInsertKey = credentials('Shredtacular-Industries-Insights-Insert-Key')
            }
            steps {
                script {
                  //Set important properties.  Change the account ID below to reflect yours.
                  nrAccountID = '1568227'
                  nrInsightsURL = "https://insights-collector.newrelic.com/v1/accounts/${nrAccountID}/events"
                  nrSyntheticsURL = 'https://synthetics.newrelic.com/synthetics/api/v4/monitors'

                  println("Begin Changesets")

                  println("End Changesets")

                  //For each Synthetics script file in the repository...
                  files = findFiles(glob: '*.js')
                    for (int i = 0; i < files.size(); i++) {
                      //Get the name of the file
                      println("Processing File: "+files[i].name)
                      monitorName = files[i].name.minus(".js")

                      //Extract the name portion - that's what the monitor's name will be.
                      println("Monitor Name: "+monitorName)
                      nameLabel = "MonitorName:${monitorName}"

                      //Get the properties file that will be used to define the job.
                      monitorProperties = readProperties file: "${monitorName}.properties"

                      //The script has to be BASE64 encoded to be uploaded.
                      nrScriptFile = readFile encoding: 'Base64', file: files[i].name
                      nrScriptFile = "{\"scriptText\": \"$nrScriptFile\"}"

                      //Construct the request body that will be used to create the monitor.
                      monitorRequestBody = "{ \"name\" : \"${monitorName}\",\"type\" : \"${monitorProperties.type}\",\"frequency\": ${monitorProperties.frequency},\"locations\": [ ${monitorProperties.locations} ],\"status\":\"${monitorProperties.status}\",\"slaThreshold\": \"${monitorProperties.slaThreshold}\" }"
                      println("Request Body: "+monitorRequestBody)

                      //Attempt to create the monitor.
                      monitorCreateResponse = httpRequest consoleLogResponseBody: true, contentType: 'APPLICATION_JSON', customHeaders: [[maskValue: false, name: 'X-Api-Key', value: nrApiKey]], httpMode: 'POST', requestBody: monitorRequestBody, validResponseCodes: '200:400', url: nrSyntheticsURL

                      //Synthetics requires that monitors have unique names, while the API requires that we know the monitor's UUID to access it.
                      //However, given the monitor name, it's not easy to query for the UUID.  It is much easier to query for a monitor's label, though.
                      // So, if the monitor doesn't exist, we create it and add a label to it that contains the monitor's name.
                      // If the monitor exists, we query for a monitor whose monitorName label matches the monitor's name.
                      // From there, we can easily get the UUID and then replace the old script with new.
                      if (monitorCreateResponse.content.matches("(.*)a monitor with that name already exists(.*)")) {
                        println("Monitor exists")
                        getLabelURL = "https://synthetics.newrelic.com/synthetics/api/v4/monitors/labels/${nameLabel}"
                        println("Request:"+getLabelURL)
                        getMonitorUUIDResponse = httpRequest consoleLogResponseBody: true, customHeaders: [[maskValue: false, name: 'X-Api-Key', value: nrApiKey]], httpMode: 'GET', url: getLabelURL
                        nrMonitorURL = (getMonitorUUIDResponse.content =~ ".*href\":\"(.*)\"")[0][1]

                        //Set the label endpoint for use later.
                        nrLabelURL = "${nrMonitorURL}/labels"

                        //Update the monitor's properties
                        monitorUpdateResponse = httpRequest consoleLogResponseBody: true, contentType: 'APPLICATION_JSON', customHeaders: [[maskValue: false, name: 'X-Api-Key', value: nrApiKey]], httpMode: 'PUT', requestBody: monitorRequestBody, url: nrMonitorURL

                        //Update the script.
                        nrScriptURL = "${nrMonitorURL}/script"
                        scriptUploadResponse = httpRequest consoleLogResponseBody: true, contentType: 'APPLICATION_JSON', customHeaders: [[maskValue: false, name: 'X-Api-Key', value: nrApiKey]], httpMode: 'PUT', requestBody: nrScriptFile, url: nrScriptURL
                      } else {
                        println("Monitor does not exist, creating new.")

                        //Upload the script, since the test has already been created.
                        newScriptURL = monitorCreateResponse.headers.Location[0]+"/script"
                        scriptUploadResponse = httpRequest consoleLogResponseBody: true, contentType: 'APPLICATION_JSON', customHeaders: [[maskValue: false, name: 'X-Api-Key', value: nrApiKey]], httpMode: 'PUT', requestBody: nrScriptFile, url: newScriptURL

                        //Set the script's name label
                        nrLabelURL = "${monitorCreateResponse.headers.Location[0]}/labels"
                        println("Label: " + nameLabel)
                        labelSetResponse = httpRequest consoleLogResponseBody: true, contentType: 'APPLICATION_JSON', customHeaders: [[maskValue: false, name: 'X-Api-Key', value: nrApiKey]], httpMode: 'POST', requestBody: nameLabel, url: nrLabelURL
                      }

                      //Set other label properties.
                      println("Setting tags")

                      //Following line removed because URLs can't be stored in labels due to character type restrictions
                      //And because managing labels is harder than I thought it'd be.
                      //nrLabelList = [ "BuildTag:${env.BUILD_TAG}", "GitCommit:${env.GIT_COMMIT}", "GitURL:${env.GIT_URL}"]
                      //nrLabelList = [ "BuildTag:${env.BUILD_TAG}", "GitCommit:${env.GIT_COMMIT}"]
                      //println("Tags = ${nrLabelList}")
                      //nrLabelList.each {
                      //  labelSetResponse = httpRequest consoleLogResponseBody: true, contentType: 'APPLICATION_JSON', customHeaders: [[maskValue: false, name: 'X-Api-Key', value: nrApiKey]], httpMode: 'POST', requestBody: it.toUpperCase(), url: nrLabelURL
                      //}

                      //Now that that's all done, let's create some custom Insights events to track what we did.
                      nrBuildEvent = "{\"eventType\":\"SyntheticsDeploymentEvent\",\"nrAccountID\": \"${nrAccountID}\",\"monitorName\": \"${monitorName}\",\"buildTag\": \"${env.BUILD_TAG}\",\"commitHash\":\"${env.GIT_COMMIT}\",\"gitURL\":\"${env.GIT_URL}\",\"buildStatus\":\"${currentBuild.currentResult}\",\"changelog\": \"\",\"description\": \"\",\"user\": \"${env.GIT_AUTHOR_EMAIL}\"}]"
                      deployEventResponse = httpRequest consoleLogResponseBody: true, contentType: 'APPLICATION_JSON', customHeaders: [[maskValue: false, name: 'X-Insert-Key', value: nrInsertKey]], httpMode: 'POST', requestBody: nrBuildEvent, url: nrInsightsURL
                    }
                }
            }
        }
    }
}
