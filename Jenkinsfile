node {
    def scmVars

    stage('build') {
      scmVars = checkout scm
      sh 'flake8 .'
    }

    stage('Deploy') {

      print "Defining vars"
      def environment = "Prod"
      def description = "Deploying my branch"
      def ref = scmVars.GIT_COMMIT
      def owner = "CommunityHoneyNetwork"
      def repo = "stingar-ci-template"
      def deployURL = "https://api.github.com/repos/${owner}/${repo}/deployments"
      def deployBody = '{"ref": "' + ref +'","environment": "' + environment  +'","description": "' + description + '"}'

      // Create new Deployment using the GitHub Deployment API
      print "Initial deploy"
      def response = httpRequest authentication: 'duke-stingar-github', httpMode: 'POST', requestBody: deployBody, responseHandle: 'STRING', url: deployURL
      if(response.status != 201) {
          error("Deployment API Create Failed: " + response.status)
      }

      print "Parsing result"
      // Get the ID of the GitHub Deployment just created
      def responseJson = readJSON text: response.content
      def id = responseJson.id
      if(id == "") {
          error("Could not extract id from Deployment response")
      }

      // Execute Deployment
      def deployStatus = sh returnStatus: true, script: 'echo deploy'

      // Record new Deployment Status based on output
      def result = (deployStatus) ? 'failure' : 'success'
      def deployStatusBody = '{"state": "' + result + '","target_url": "http://github.com/deploymentlogs"}'
      def deployStatusURL = "https://api.github.com/repos/${owner}/${repo}/deployments/${id}/statuses"
      def deployStatusResponse = httpRequest authentication: 'duke-stingar-github', httpMode: 'POST', requestBody: deployStatusBody , responseHandle: 'STRING', url: deployStatusURL
      if(deployStatusResponse.status != 201) {
        error("Deployment Status API Update Failed: " + deployStatusResponse.status)
      }
   }
}
