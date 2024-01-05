---
layout: post
title: Restarting Octopus Deploy Tentacle during a deployment
tags: powershell octopus
---
When deploying an Octopus project, one of your steps may involve editing or creating an environment variable. One of the most common variables to be updated when installing an application is the PATH variable. An installation adds the folder where the application can be found to the PATH variable. 
In other cases, an application will create a new environment variable. Java, for example, will create a new variable called JAVA_HOME, which points to the folder where the JDK files can be found. 
Environment variables are loaded when the tentacle process is started, and the child processes (Calamari, PowerShell) inherit the environment variables from the parent process. This means that newly created or updated variables won't be found until the tentacle process is restarted or manually loaded into your current context.
One way around this would be to manually load or update the variable using `[System.Environment]::GetEnvironmentVariable(VariableName, EnvironmentVariableTarget)` and assign it to an environment variable (`$Env:VariableName`) in the current scope. 
Here's an example for JAVA_HOME.
```powershell
$env:JAVA_HOME = [System.Environment]::GetEnvironmentVariable('JAVA_HOME', 'machine')
```
While this will work if you need to access the variable in the current step, the downside is that it won't exist in any subsequent steps or deployment runs.

The way around this is to restart the tentacle on the target machine. After the restart, the tentacle and its children will know whether the new environment variables have been set. The problem is, that if we restart the tentacle in a deployment step with `restart-service OctopusDeploy Tentacle: tentacle`, the step will fail because we're killing its own process and the deployment will fail. 
![PsConf](/assets/img/20230829120331.png)
So if we want to restart the tentacle service in one step without failing the whole deployment, how do we achieve the restart?

First, we create a new step that runs on a worker on behalf of a target. This step can then be extracted as a step template and used in any deployment or runbook. To ensure that the step is running "on behalf of", we will check the context in which this step is running. 
``` powershell
if($OctopusParameters["Octopus.Action.TargetRoles"] -eq '' -or $OctopusParameters["Octopus.Action.RunOnServer"] -eq "false"){
	Throw "The step can only 'run on behalf of'"
}
```
The next step is to run the restart service on the actual target machine. We can do this by initiating an ad hoc task from within our step. This can be done using the REST API or the .net octopus client. For this example we'll use the REST API.

As we know, we cannot simply run `Restart Serverice 'OctopusDeploy Tentacle:*'` as this will cause the ad hoc script to fail. Instead, we'll start a new PowerShell process and wait a few seconds before restarting the service. This ensures that the parent process has finished successfully before the actual service restart is initiated. This is what the AdHoc script will look like: `Start-Process powershell.exe -WorkingDirectory $env:TEMP -ArgumentList "-Command "&{Start-Sleep 3 ; Get-Service ''OctopusDeploy Tentacle:*'' | Restart-Service}"`.

Now all we need to do is run the ad hoc script from our worker:
```powershell
$machineID = $OctopusParameters["Octopus.Machine.Id"]
$machineName = $OctopusParameters["Octopus.Machine.Name"]
$EnvironmentID = $OctopusParameters["Octopus.Environment.Id"]
$spaceId = $OctopusParameters["Octopus.Space.Id"]
$serverUrl = $OctopusParameters["Octopus.Web.BaseUrl"]

$header = @{ "X-Octopus-ApiKey" = $API_KEY } # $API_KEY is a step parameter 

$script = 'Start-Process powershell.exe -WorkingDirectory $env:TEMP -ArgumentList "-Command  `"&{Start-Sleep 3 ; Get-Service ''OctopusDeploy Tentacle:*'' | Restart-Service}`""'

$arguments = @{
    MachineIds = @($machineID)
    TargetType = "Machines"
    Syntax = "Powershell"
    ScriptBody = $script
}

# Create Payload
$scriptTaskBody = (@{
    Name = "AdHocScript"
    Description = "Restarting Tentacle on $machineName - $machineID"
    Arguments = $arguments
    SpaceId = $spaceId
}) | ConvertTo-Json -Depth 10

# Run AdHocScript
$task = Invoke-RestMethod -Method "POST" "$($serverUrl)/api/tasks" -body $scriptTaskBody -Headers $header -ContentType "application/json"
```

Next, we want to check that the ad hoc script step was successful. We'll continue to monitor the task until it completes, and after the task completes, we'll check to see if it completed successfully.
```powershell
# Check if task is finished
$task = Invoke-RestMethod -Method "GET" "$($serverUrl)/api/tasks/$($task.Id)" -Headers $header -ContentType "application/json"

while ($task.IsCompleted -eq $false) {
        Write-Host "Waiting for task to finish..."
        Start-Sleep -Seconds 5
        $task = Invoke-RestMethod -Method "GET" "$($serverUrl)/api/tasks/$($task.Id)" -Headers $header -ContentType "application/json"
}

# Check if task was successful
if ($task.State -ne "Success") {
    Write-Error "Restart Tentacle Failed" -ErrorAction Stop
}
```

Now that the task has been successfully completed, we want to wait a few seconds for the actual reboot to take place. In most cases waiting 30 seconds is sufficient, but on slow machines it may take longer. To be absolutely sure that our restart was successful and that the tentacle is ready to receive new tasks, we want to run health checks until successful. A health check, or any job for that matter, can fail if it is run while the tentacle is initialising. Therefore, we will try to run a health check for up to 3 minutes. If it fails within the 3 minutes, we'll try again.
``` powershell
# Wait for restart of Tentacle
Write-Output "Restart executed"
Start-Sleep 30

# Check if Tentacle is communicating again
Write-Host "Checking Tentacle is communicating again."

# Create health check task
$healthCheckBody = (@{
        Name        = "Health"
        Description = "Health check for $machineName - $machineID"
        SpaceId     = $spaceId
        Arguments   = @{
            Timeout        = "$([TimeSpan]::FromMinutes(1))"
            MachineTimeout = "$([TimeSpan]::FromMinutes(1))"
            EnvironmentId  = $EnvironmentID
            MachineIds     = @($MachineId)
        }
    }) | ConvertTo-Json -Depth 10

# Create a loop that loops for 3 minutes and checks if the tentacle is communicating again
$timeout = 180 #restart is allowed to take max 3min
$sw = [System.Diagnostics.Stopwatch]::StartNew()

do {
    # Create health check task
    $task = Invoke-RestMethod -Method "Post" -Uri "$serverUrl/api/$($spaceId)/tasks" -Body $healthCheckBody -Headers $header
    Write-Highlight "Tentacle Health Check Task: [here]($($serverUrl)$($task.links.web))"
    Write-Output "Health check task created."

    # Wait for health check task to finish
    while ($task.IsCompleted -eq $false) {
        Write-Host "Waiting for health check to finish..."
        Start-Sleep -Seconds 5
        $task = Invoke-RestMethod -Method "GET" "$($serverUrl)/api/tasks/$($task.Id)" -Headers $header -ContentType "application/json"
    }

        # Check if finished task was successful
        if ($task.State -eq "Success") {
            Write-Output "Health check finished successfully."
            break
        }
        else {
            Write-Output "Health check failed. Checking timeout before creating new health check task."
        }
} while ($sw.IsRunning -and $sw.Elapsed.TotalSeconds -lt $timeout)
```

Finally, we'll check that the last health check was successful, as we don't yet know whether the loop ended because of the 3-minute timeout or because the health check was successful.
```powershell
# Final check if tentacle is communicating again

if ($task.State -ne "Success") {
    Write-Error "Tentacle is not communicating again after restart." -ErrorAction Stop
}
else {
    Write-Output "Tentacle is communicating again."
}
```

The script is finished and you can now add the script as a step to your project and a tentacle will be restarted during the deployment. I hope this article helps you with restarting a tentacle during deployments. 
Here's the full script:
```powershell
# Check that this step only runs on the server "on behalf of"

if($OctopusParameters["Octopus.Action.TargetRoles"] -eq '' -or $OctopusParameters["Octopus.Action.RunOnServer"] -eq "false"){
    Throw "The step can only 'run on behalf of'"
}

# Setting up variables
$machineID = $OctopusParameters["Octopus.Machine.Id"]
$machineName = $OctopusParameters["Octopus.Machine.Name"]
$EnvironmentID = $OctopusParameters["Octopus.Environment.Id"]
$spaceId = $OctopusParameters["Octopus.Space.Id"]
$serverUrl = $OctopusParameters["Octopus.Web.BaseUrl"]
$header = @{ "X-Octopus-ApiKey" = $API_KEY }

$script = 'Start-Process powershell.exe -WorkingDirectory $env:TEMP -ArgumentList "-Command  `"&{Start-Sleep 3 ; Get-Service ''OctopusDeploy Tentacle:*'' | Restart-Service}`""'

# Create Payload
$scriptTaskBody = (@{
        Name        = "AdHocScript"
        Description = "Restarting Tentacle on $machineName - $machineID"
        SpaceId     = $spaceId
        Arguments    = @{
            MachineIds = @($machineID)
            TargetType = "Machines"
            Syntax     = "Powershell"
            ScriptBody = $script
       }
    }) | ConvertTo-Json -Depth 10

# Run AdHocScript
$task = Invoke-RestMethod -Method "POST" "$($serverUrl)/api/tasks" -Body $scriptTaskBody -Headers $header -ContentType "application/json"

# Check if task is finished
$task = Invoke-RestMethod -Method "GET" "$($serverUrl)/api/tasks/$($task.Id)" -Headers $header -ContentType "application/json"
Write-Output "Restart initiated"
Write-Highlight "Restart Tentacle Task: [here]($($serverUrl)$($task.links.web))"

while ($task.IsCompleted -eq $false) {
    Write-Host "Waiting for restart task to finish..."
    Start-Sleep -Seconds 5
    $task = Invoke-RestMethod -Method "GET" "$($serverUrl)/api/tasks/$($task.Id)" -Headers $header -ContentType "application/json"
}  

# Check if task was successful
if ($task.State -ne "Success") {
        Write-Error "Restart Tentacle Failed" -ErrorAction Stop
}  

# Wait for restart of Tentacle
Write-Output "Restart executed"
Start-Sleep 30

# Check if Tentacle is communicating again
Write-Host "Checking Tentacle is communicating again."

# Create health check task
$healthCheckBody = (@{
    Name        = "Health"
    Description = "Health check for $machineName - $machineID"
    SpaceId     = $spaceId
    Arguments   = @{
        Timeout        = "$([TimeSpan]::FromMinutes(1))"
        MachineTimeout = "$([TimeSpan]::FromMinutes(1))"
        EnvironmentId  = $EnvironmentID
        MachineIds     = @($MachineId)
    }
}) | ConvertTo-Json -Depth 10

# Create a loop that loops for 3 minutes and checks if the tentacle is communicating again
$timeout = 180 #restart is allowed to take max 3min
$sw = [System.Diagnostics.Stopwatch]::StartNew()
do {
    # Create health check task
    $task = Invoke-RestMethod -Method "Post" -Uri "$serverUrl/api/$($spaceId)/tasks" -Body $healthCheckBody -Headers $header
    Write-Highlight "Tentacle Health Check Task: [here]($($serverUrl)$($task.links.web))"
    Write-Output "Health check task created." 

    # Wait for health check task to finish
    while ($task.IsCompleted -eq $false) {
        Write-Host "Waiting for health check to finish..."
        Start-Sleep -Seconds 5
        $task = Invoke-RestMethod -Method "GET" "$($serverUrl)/api/tasks/$($task.Id)" -Headers $header -ContentType "application/json"
    }

    # Check if finished task was successful
    if ($task.State -eq "Success") {
        Write-Output "Health check finished successfully."
        break
    }else {
        Write-Output "Health check failed. Checking timeout before creating new health check task."
    }
} while ($sw.IsRunning -and $sw.Elapsed.TotalSeconds -lt $timeout)

# Final check if tentacle is communicating again
if ($task.State -ne "Success") {
    Write-Error "Tentacle is not communicating again after restart." -ErrorAction Stop
}else {
    Write-Output "Tentacle is communicating again."
}
```