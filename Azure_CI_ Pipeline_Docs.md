
# 1. Create an Organization 
# 2. Create a Project 
# 3. Clone a repository 
# 4. Create a pipeline
# 5. Create a Container Registry in Azure Portal
- Sample pipeline azure-pipelines.yaml file
```yaml
    
    # Custom Path instead of Continously Integrating entire repo. 
    trigger:
      paths:
        include:
          - result/*

    resources:
    - repo: self

    variables:
      # Container registry service connection established during pipeline creation
      dockerRegistryServiceConnection: '432620be-a271-449b-b563-bf67e49f9c28'
      imageRepository: 'resultservice'
      containerRegistry: 'azcicdtesting.azurecr.io'
      dockerfilePath: '$(Build.SourcesDirectory)/result/Dockerfile'
      tag: '$(Build.BuildId)'

      # Agent VM image name
      # vmImageName: 'ubuntu-latest'
     
    # Below is the Self hosted agent because free trail doesnot support Microsoft managed agents. 
    pool: 
      name: 'myvm'
    stages:
    - stage: BuildandPush
      displayName: Build and Push stage
      jobs:
      - job: BuildandPush
        displayName: Build and Push
        steps:
        - task: Docker@2
          displayName: Build and Push an image to container registry
          inputs:
            containerRegistry: '$(dockerRegistryServiceConnection)'
            repository: '$(imageRepository)'
            command: 'buildAndPush'
            Dockerfile: 'result/Dockerfile'
            tags: '$(tag)'
            arguments: '--no-cache --platform linux/arm64'

    - stage: Update
      displayName: Update stage
      jobs:
      - job: Update
        displayName: Update
        steps:
        - task: ShellScript@2
          inputs:
            scriptPath: 'scripts/updateK8sManifest.sh'
            args: 'result $(imageRepository) $(tag)'
```

# 6. Create an AgentPool
- Goto "Project settings" on bottom left> "Agent pools" > "Add Pool" on top right> Choose the "Pool Type" based on the requirement
- Chosen self hosted linux agent.
- Create an Agent Pool as same name as given in the above ci pipeline maifestation file. 'myvm' in this case

# 7. Create a VM for Self Hosted Agent
- A basic VM creation.

# 8. Create Agent in the Agent Pool
- Goto previously created agent pool.
- Create new agent
```bash
    mkdir myagent
    cd myagent
    wget https://download.agent.dev.azure.com/agent/4.270.0/vsts-agent-linux-x64-4.270.0.tar.gz
    tar zxvf vsts-agent-linux-x64-4.270.0.tar.gz
```
# 9. Configure the Agent in the VM
``` bash
    cd myagent # previously created directory 
    ./config.sh
    # follow the steps for configuration use Personal Acess Tokens as Authorization purpose.
```
- server url: https://dev.azure.com/{your-organization}
- Docs URL: https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/linux-agent?view=azure-devops&tabs=IP-V4
# 10. Run the Agent in the VM
```bash
    cd myagent
    ./run.sh
```
- Now the Agent is ready to implement CI.
- we can now run pipelines from AZ devops portal.
