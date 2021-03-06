```powershell PowerShell (REST API)
$ErrorActionPreference = "Stop";

# Define working variables
$octopusURL = "https://youroctourl"
$octopusAPIKey = "API-YOURAPIKEY"
$header = @{ "X-Octopus-ApiKey" = $octopusAPIKey }
$spaceName = "default"
$role = "My role"
$scriptBody = "Write-Host `"Hello world`""

# Project details
$projectName = "MyProject"


# Get space
$space = (Invoke-RestMethod -Method Get -Uri "$octopusURL/api/spaces/all" -Headers $header) | Where-Object {$_.Name -eq $spaceName}

# Get project
$project = (Invoke-RestMethod -Method Get -Uri "$octopusURL/api/$($space.Id)/projects/all" -Headers $header) | Where-Object {$_.Name -eq $projectName}

# Get deployment process
$deploymentProcess = (Invoke-RestMethod -Method Get -Uri "$octopusURL/api/$($space.Id)/deploymentprocesses/$($project.DeploymentProcessId)" -Headers $header)

# Get current steps
$steps = $deploymentProcess.Steps

# Add new step to process
$steps += @{
    Name = "Run a script"
    Properties = @{
    'Octopus.Action.TargetRoles' = $role
    }
    Condition = "Success"
    StartTrigger = "StartAfterPrevious"
    PackageRequirement = "LetOctopusDecide"
    Actions = @(
    @{
        ActionType = "Octopus.Script"
        Name = "Run a script"
        Environments = @()
        ExcludedEnvironments = @()
        Channels = @()
        TenantTags = @()
        Properties = @{
            'Octopus.Action.RunOnServer' = "false"
            'Octopus.Action.EnabledFeatures' = ""
            'Octopus.Action.Script.ScriptSource' = "Inline"
            'Octopus.Action.Script.Syntax' = "PowerShell"
            'Octopus.Action.Script.ScriptFilename' = $null
            'Octopus.Action.Script.ScriptBody' = $scriptBody
        }
        Packages = @()
        IsDisabled = $false
        WorkerPoolId = ""
        WorkerPoolVariable = ""
        Container = @{
            "FeedId" = $null
            "Image" = $null
        }
        CanBeUsedForProjectVersioning = $false
        IsRequired = $false
    }
    )
}

# Convert steps to json
$deploymentProcess.Steps = $steps
$jsonPayload = $deploymentProcess | ConvertTo-Json -Depth 10

# Submit request
Invoke-RestMethod -Method Put -Uri "$octopusURL/api/$($space.Id)/deploymentprocesses/$($project.DeploymentProcessId)" -Headers $header -Body $jsonPayload
```
```powershell PowerShell (Octopus.Client)
# Load Octopous Client assembly
Add-Type -Path 'c:\octopus.client\Octopus.Client.dll'

# Declare Octopus variables
$apikey = 'API-YOURAPIKEY'
$octopusURI = 'https://youroctourl'
$projectName = "MyProject"

# Create repository object
$endpoint = New-Object Octopus.Client.OctopusServerEndpoint $octopusURI,$apikey
$repository = New-Object Octopus.Client.OctopusRepository $endpoint
$client = New-Object Octopus.Client.OctopusClient($endpoint)

# Get reference to space
$space = $repository.Spaces.FindByName($spaceName)
$repositoryForSpace = $client.ForSpace($space)

try
{
    # Get project
    $project = $repositoryForSpace.Projects.FindByName($projectName)

    # Get project process
    $process = $repositoryForSpace.DeploymentProcesses.Get($project.DeploymentProcessId)

    # Define new step
    $stepName = "Run a script" # The name of the step
    $role = "My role" # The machine role to run this step on
    $scriptBody = "Write-Host 'Hello world'" # The script to run
    $step = New-Object Octopus.Client.Model.DeploymentStepResource
    $step.Name = $stepName
    $step.Condition = [Octopus.Client.Model.DeploymentStepCondition]::Success
    $step.Properties.Add("Octopus.Action.TargetRoles", $role)

    # Define script action
    $scriptAction = New-Object Octopus.Client.Model.DeploymentActionResource
    $scriptAction.ActionType = "Octopus.Script"
    $scriptAction.Name = $stepName
    $scriptAction.Properties.Add("Octopus.Action.Script.ScriptBody", $scriptBody)

    # Add step to process
    $step.Actions.Add($scriptAction)
    $process.Steps.Add($step)
    $repositoryForSpace.DeploymentProcesses.Modify($process)
}
catch
{
    # Display error
    Write-Host $_.Exception.Message
}
```
```csharp C#
// If using .net Core, be sure to add the NuGet package of System.Security.Permissions
#r "path\to\Octopus.Client.dll"

using Octopus.Client;
using Octopus.Client.Model;

// Declare working varibles
var octopusURL = "http://OctoTemp";
var octopusAPIKey = "API-YOURAPIKEY";
string stepName = "Run a script";
string roleName = "My role";
string scriptBody = "Write-Host \"Hello world\"";
string projectName = "MyProject";
string spaceName = "default";

// Create repository object
var endpoint = new OctopusServerEndpoint(octopusURL, octopusAPIKey);
var repository = new OctopusRepository(endpoint);
var client = new OctopusClient(endpoint);

try
{
    // Get space
    var space = repository.Spaces.FindByName(spaceName);
    var repositoryForSpace = client.ForSpace(space);

    // Get project
    var project = repositoryForSpace.Projects.FindByName(projectName);

    // Get the deployment process
    var deploymentProcess = repositoryForSpace.DeploymentProcesses.Get(project.DeploymentProcessId);

    // Create new step object
    Octopus.Client.Model.DeploymentStepResource newStep = new DeploymentStepResource();
    newStep.Name = stepName;
    newStep.Condition = DeploymentStepCondition.Success;
    newStep.Properties.Add("Octopus.Action.TargetRoles", roleName);

    // Create new script action
    Octopus.Client.Model.DeploymentActionResource stepAction = new DeploymentActionResource();
    stepAction.ActionType = "Octopus.Script";
    stepAction.Name = stepName;
    stepAction.Properties.Add("Octopus.Action.Script.ScriptBody", scriptBody);

    // Add step action and step to process
    newStep.Actions.Add(stepAction);
    deploymentProcess.Steps.Add(newStep);

    // Update process
    repositoryForSpace.DeploymentProcesses.Modify(deploymentProcess);
}
catch (Exception ex)
{
    Console.WriteLine(ex.Message);
    return;
}
```
```python Python3
import json
import requests

octopus_server_uri = 'https://your.octopus.app/api'
octopus_api_key = 'API-YOURAPIKEY'
headers = {'X-Octopus-ApiKey': octopus_api_key}


def get_octopus_resource(uri):
    response = requests.get(uri, headers=headers)
    response.raise_for_status()

    return json.loads(response.content.decode('utf-8'))


space_name = 'Default'
project_name = 'Your project name'
role_name = 'Your target role'
step_name = 'New run a script step'

spaces = get_octopus_resource('{0}/spaces/all'.format(octopus_server_uri))
space = next((x for x in spaces if x['Name'] == space_name), None)

projects = get_octopus_resource('{0}/{1}/projects/all'.format(octopus_server_uri, space['Id']))
project = next((x for x in projects if x['Name'] == project_name), None)

uri = '{0}/{1}/deploymentprocesses/{2}'.format(octopus_server_uri, space['Id'], project['DeploymentProcessId'])
process = get_octopus_resource(uri)

process['Steps'].append({
    'Name': step_name,
    'Properties': {
        'Octopus.Action.TargetRoles': role_name
    },
    'Condition': 'Success',
    'StartTrigger': 'StartAfterPrevious',
    'PackageRequirement': 'LetOctopusDecide',
    'Actions': [{
        'ActionType': 'Octopus.Script',
        'WorkerPoolId': None,
        'Container': {'Image': None, 'FeedId': None},
        'WorkerPoolVariable': None,
        'Name': step_name,
        'Environments': [],
        'Channels': [],
        'TenantTags': [],
        'Properties': {
            'Octopus.Action.RunOnServer': 'false',
            'Octopus.Action.EnabledFeatures': '',
            'Octopus.Action.Script.ScriptSource': 'Inline',
            'Octopus.Action.Script.Syntax': 'Python',
            'Octopus.Action.Script.ScriptFilename': None,
            'Octopus.Action.Script.ScriptBody': 'print("Hello world")'
        }
    }]
})

response = requests.put(uri, headers=headers, json=process)
response.raise_for_status()
```
