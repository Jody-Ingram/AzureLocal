function Invoke-AzureLocalForceClusterDeployment {
<#
.SYNOPSIS
Starts an Azure Local cluster ARM deployment with retry logic for known timeout or cancellation failure patterns.

.DESCRIPTION
The Invoke-AzureLocalForceClusterDeployment function deploys an Azure Local cluster by running an ARM template
deployment at subscription scope.

The function monitors the deployment until it reaches a terminal state. If the deployment fails with a known
Azure Local timeout, cancellation, CloudDeploy, Arc notification, or task cancellation pattern, the function can
retry the deployment using a new deployment name for each attempt.

.PARAMETER SubscriptionNameOrId
The Azure subscription name or subscription ID where the Azure Local deployment will run.

.PARAMETER Location
The Azure region used for the subscription-scope deployment metadata.

.PARAMETER TemplateFile
The local path to the Azure Local ARM template JSON file.

.PARAMETER TemplateParameterFile
The local path to the Azure Local ARM template parameters JSON file.

.PARAMETER DeploymentName
The base deployment name. Each retry appends -try1, -try2, etc. to keep deployment history clean.

.PARAMETER MaxRetries
The maximum number of deployment attempts.

.PARAMETER PollSeconds
The number of seconds to wait between deployment status checks.

.PARAMETER RetryDelaySeconds
The number of seconds to wait before retrying after a recognized timeout or cancellation pattern.

.EXAMPLE
Invoke-AzureLocalForceClusterDeployment `
    -SubscriptionNameOrId "SUBSCRIPTION_NAME" `
    -Location "eastus" `
    -TemplateFile "C:\Temp\azurelocal-template.json" `
    -TemplateParameterFile "C:\Temp\azurelocal-parameters.json"

Runs the Azure Local deployment and retries recognized timeout-style failures.

.EXAMPLE
Invoke-AzureLocalForceClusterDeployment `
    -SubscriptionNameOrId "SUBSCRIPTION_NAME" `
    -TemplateFile "C:\Azure\azurelocalTemplate.json" `
    -TemplateParameterFile "C:\Azure\azurelocalParameters.json" `
    -MaxRetries 5 `
    -PollSeconds 120 `
    -Verbose

Runs the deployment with five attempts and checks deployment status every two minutes.

.NOTES
Script  : AzureLocal-ForceClusterDeployment.ps1
Version : 1.0
Author  : Jody Ingram
Purpose : Deploys Azure Local Cluster using ARM template with custom retry logic for common timeout/cancel failure patterns.
#>

    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true)]
        [ValidateNotNullOrEmpty()]
        [string]$SubscriptionNameOrId,

        [Parameter(Mandatory = $false)]
        [ValidateNotNullOrEmpty()]
        [string]$Location = "eastus",

        [Parameter(Mandatory = $true)]
        [ValidateScript({
            if (-not (Test-Path -Path $_ -PathType Leaf)) {
                throw "Template file not found: $_"
            }

            return $true
        })]
        [string]$TemplateFile,

        [Parameter(Mandatory = $true)]
        [ValidateScript({
            if (-not (Test-Path -Path $_ -PathType Leaf)) {
                throw "Template parameter file not found: $_"
            }

            return $true
        })]
        [string]$TemplateParameterFile,

        [Parameter(Mandatory = $false)]
        [ValidateNotNullOrEmpty()]
        [string]$DeploymentName = ("azurelocalDeploy-" + (Get-Date -Format "yyyyMMdd-HHmm")),

        [Parameter(Mandatory = $false)]
        [ValidateRange(1, 25)]
        [int]$MaxRetries = 3,

        [Parameter(Mandatory = $false)]
        [ValidateRange(15, 3600)]
        [int]$PollSeconds = 60,

        [Parameter(Mandatory = $false)]
        [ValidateRange(0, 7200)]
        [int]$RetryDelaySeconds = 300
    )

    begin {
        $ErrorActionPreference = "Stop"

        function Write-Log {
            [CmdletBinding()]
            param(
                [Parameter(Mandatory = $true)]
                [string]$Message
            )

            Write-Host ("[{0}] {1}" -f (Get-Date -Format "yyyy-MM-dd HH:mm:ss"), $Message)
        }

        function Import-RequiredModule {
            [CmdletBinding()]
            param(
                [Parameter(Mandatory = $true)]
                [string[]]$Name
            )

            foreach ($moduleName in $Name) {
                Write-Verbose "Importing module '$moduleName'."
                Import-Module $moduleName -ErrorAction Stop
            }
        }

        function Test-TimeoutLikeFailure {
            [CmdletBinding()]
            param(
                [Parameter(Mandatory = $false)]
                [object]$Deployment,

                [Parameter(Mandatory = $false)]
                [object]$Operations
            )

            if (-not $Deployment) {
                return $false
            }

            $raw = @(
                $Deployment | ConvertTo-Json -Depth 20

                if ($Operations) {
                    $Operations | ConvertTo-Json -Depth 20
                }
            ) -join "`n"

            return (
                $raw -match "No Updates were received from the HCI device in the last 60 minutes" -or
                $raw -match "No updates received for 60 minutes" -or
                $raw -match "CleanStuckJobInProgress" -or
                $raw -match "CloudDeploy_Deploy Operation cancelled" -or
                $raw -match "DeployClusterOperationFailed" -or
                $raw -match "PostArcNotificationFailed" -or
                $raw -match "HttpClient.Timeout" -or
                $raw -match "Operation cancelled" -or
                $raw -match "task was canceled"
            )
        }

        function Get-DeploymentTerminalState {
            [CmdletBinding()]
            param(
                [Parameter(Mandatory = $true)]
                [string]$Name
            )

            Get-AzSubscriptionDeployment -Name $Name -ErrorAction Stop
        }

        function Start-AzureLocalDeployment {
            [CmdletBinding()]
            param(
                [Parameter(Mandatory = $true)]
                [string]$Name,

                [Parameter(Mandatory = $true)]
                [string]$DeploymentLocation,

                [Parameter(Mandatory = $true)]
                [string]$TemplatePath,

                [Parameter(Mandatory = $true)]
                [string]$ParameterPath
            )

            Write-Log "Starting subscription deployment '$Name' in location '$DeploymentLocation'."

            $null = New-AzSubscriptionDeployment `
                -Name $Name `
                -Location $DeploymentLocation `
                -TemplateFile $TemplatePath `
                -TemplateParameterFile $ParameterPath `
                -Verbose `
                -AsJob

            Write-Log "Deployment submitted."
        }

        function Wait-AzureLocalDeployment {
            [CmdletBinding()]
            param(
                [Parameter(Mandatory = $true)]
                [string]$Name,

                [Parameter(Mandatory = $true)]
                [int]$PollIntervalSeconds
            )

            while ($true) {
                Start-Sleep -Seconds $PollIntervalSeconds

                try {
                    $deployment = Get-DeploymentTerminalState -Name $Name
                }
                catch {
                    Write-Log "Could not read deployment state. Retrying state check."
                    continue
                }

                Write-Log ("ProvisioningState = {0}" -f $deployment.ProvisioningState)

                switch ($deployment.ProvisioningState) {
                    "Succeeded" { return $deployment }
                    "Failed"    { return $deployment }
                    "Canceled"  { return $deployment }
                    default     { }
                }
            }
        }

        function Get-DeploymentFailureDetails {
            [CmdletBinding()]
            param(
                [Parameter(Mandatory = $true)]
                [string]$Name
            )

            try {
                $operations = Get-AzSubscriptionDeploymentOperation `
                    -DeploymentName $Name `
                    -ErrorAction Stop

                return $operations | Sort-Object Timestamp -Descending
            }
            catch {
                Write-Log "Unable to retrieve deployment operations."
                return $null
            }
        }

        Import-RequiredModule -Name @(
            "Az.Accounts",
            "Az.Resources"
        )
    }

    process {
        $attempt = 1
        $lastFailure = $null

        Write-Log "Connecting to Azure."
        Connect-AzAccount

        Write-Log "Setting Azure context to subscription '$SubscriptionNameOrId'."
        Set-AzContext -Subscription $SubscriptionNameOrId -ErrorAction Stop | Out-Null

        while ($attempt -le $MaxRetries) {
            $currentDeploymentName = "{0}-try{1}" -f $DeploymentName, $attempt

            Write-Log "=== Attempt $attempt of $MaxRetries ==="

            Start-AzureLocalDeployment `
                -Name $currentDeploymentName `
                -DeploymentLocation $Location `
                -TemplatePath $TemplateFile `
                -ParameterPath $TemplateParameterFile

            $result = Wait-AzureLocalDeployment `
                -Name $currentDeploymentName `
                -PollIntervalSeconds $PollSeconds

            if ($result.ProvisioningState -eq "Succeeded") {
                Write-Log "Deployment succeeded on attempt $attempt."
                return $result
            }

            Write-Log "Deployment ended with state '$($result.ProvisioningState)' on attempt $attempt."
            $lastFailure = $result

            $operations = Get-DeploymentFailureDetails -Name $currentDeploymentName

            if ($operations) {
                Write-Log "Recent deployment operation details:"

                $operations |
                    Select-Object -First 10 OperationId, ProvisioningState, Timestamp, TargetResource, StatusCode, StatusMessage |
                    Format-List
            }

            $timeoutLike = Test-TimeoutLikeFailure `
                -Deployment $result `
                -Operations $operations

            if ($timeoutLike -and $attempt -lt $MaxRetries) {
                Write-Log "Detected timeout/cancel pattern. Waiting $RetryDelaySeconds seconds before retry."
                Start-Sleep -Seconds $RetryDelaySeconds

                $attempt++
                continue
            }

            Write-Log "Failure was not recognized as auto-retryable, or max retries reached."
            break
        }

        Write-Error "Deployment did not succeed after $attempt attempt(s)."

        if ($lastFailure) {
            $lastFailure | Format-List *
        }
    }
}