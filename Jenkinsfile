node {
    def commit_id
    stage('Cleanup'){
      cleanWs cleanWhenFailure: true, deleteDirs: true, disableDeferredWipeout: false
    }
    stage('Preparation') {
      checkout scm
      sh "git rev-parse --short HEAD > .git/commit-id"                        
      commit_id = readFile('.git/commit-id').trim()
    }  
    stage('Docker build/push') {     
      docker.withRegistry('https://index.docker.io/v1/', 'docker-gurpritsingh') {
        def app = docker.build("gurpritsingh/bufalo-fullstack:${env.BUILD_NUMBER}", '.').push()
      }
      docker.withRegistry('https://index.docker.io/v1/', 'docker-gurpritsingh') {
        def app = docker.build("gurpritsingh/bufalo-fullstack:${commit_id}", '.').push()
      }
    }
    stage('Db Creation') {     
      withCredentials([string(credentialsId: 'databasePassword', variable: 'dbPass')]) {
        sh"""
        cd dbCreation
        docker-compose up -d
        sleep 30
        """
        }
    }
    stage('Deploy to App Service') {
        withCredentials([azureServicePrincipal('Devops-free-trial-subs'), string(credentialsId: 'docker-gurpritsingh-password', variable: 'dockerPassword'), string(credentialsId: 'databasePassword', variable: 'dbPass')]) {
          echo "Deploying to azure app service"
          sh """
              cd armTemplates
              resourceGroup='devops-pj-dev'
              deploymentName='bufalo-fullstack'
              az login --service-principal --username ${AZURE_CLIENT_ID} --password ${AZURE_CLIENT_SECRET} --tenant ${AZURE_TENANT_ID} # --subscription ${AZURE_SUBSCRIPTION_ID}
              az group create -l eastus2 -n \$resourceGroup --subscription ${AZURE_SUBSCRIPTION_ID}
              az deployment group create --resource-group \$resourceGroup  --name \$deploymentName --template-file azureDeploy.json --parameters dev.parameters.json --parameters dockerRegistryPassword=${dockerPassword} --parameters linuxFxVersion="DOCKER|gurpritsingh/bufalo-fullstack:${env.BUILD_NUMBER}" --parameters databasePassword=$dbPass --parameters dockerRegistryPassword=${dockerPassword} --parameters databaseUrl='postgres://postgres:$dbPass@20.235.66.219:5432/gobuff_realworld_example_app_production?sslmode=disable' --parameters testDatabaseUrl='postgres://postgres:$dbPass@20.235.66.219:5432/gobuff_realworld_example_app_production?sslmode=disable' --parameters databaseHost='20.235.66.219' --parameters waitDbHosts='20.235.66.219:5432'
            """
          }
      }
      stage('Post Cleanup'){
          cleanWs cleanWhenFailure: true, deleteDirs: true, disableDeferredWipeout: false
    }
  }