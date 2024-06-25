---
layout: post
title:  "Adding Members to Microsoft Teams Channels by Tag Using PowerShell"
date:   2024-06-25 06:00:05 +0100
tags: [azure, ms-teams, powershell-script, productivity] 
---

## Overview

I was working on a real use case scenario to add a large list of users to multiple private channels in MS Teams.
The first option that came to my mind was adding them manually by adding each user to each channel, but after some thought, I asked myself some questions:

1. Doing it manually will be time-consuming? The answer was "yes" as it's a large list and it involves not just one channel but several channels.
2. Is this a repetitive task? The answer was yes as new members are constantly joining our team. Additionally, if we create a new channel and want to add these members to the newly created one we will have to repeat the process.

Therefore, doing it manually will be time-consuming and this is a repeated task, that is why I thought about another solution to be more productive.

Since these users are assigned tags, what if we add them to the channels based on their tags? I couldn't find this option in MS Teams. This led me to think about developing a script that automate the process of adding users to the channels based on their tags.

All you need is to provide the Team link, channel link, and the tag name. The script will list all users that are assigned to the provided tag and add them to the specified channel.

## Script

```powershell
# Function to load configuration from a JSON file
function Import-Config {
    param (
        [string]$configPath
    )
    if (Test-Path $configPath) {
        return Get-Content -Path $configPath | ConvertFrom-Json
    } else {
        Write-Host "Configuration file not found: $configPath"
        exit
    }
}

# Load configuration
$configPath = ".\config.json"
$config = Import-Config -configPath $configPath

# Validate configuration
if (-not $config.tenantId -or -not $config.clientId -or -not $config.clientSecret -or -not $config.batchSize -or -not $config.batchDelay) {
    Write-Host "Configuration file is missing required parameters."
    exit
}

$tenantId = $config.tenantId
$clientId = $config.clientId
$clientSecret = $config.clientSecret
$batchSize = $config.batchSize
$batchDelay = $config.batchDelay

# Function to extract teamId from URL
function Get-TeamIdFromUrl {
    param (
        [ValidateNotNullOrEmpty()]
        [string]$url
    )
    if ($url -match "groupId=([a-f0-9-]+)") {
        return $matches[1]
    } else {
        Write-Host "Invalid URL. Could not extract teamId."
        return $null
    }
}

# Function to extract channelId from URL
function Get-ChannelIdFromUrl {
    param (
        [ValidateNotNullOrEmpty()]
        [string]$url
    )
    if ($url -match "channel/([^/]+)") {
        return $matches[1]
    } else {
        Write-Host "Invalid URL. Could not extract channelId."
        return $null
    }
}

# Get the team URL from user input
$teamUrl = Read-Host "Enter the Team Link: Go to your team, click on the three dots ..., click on Get a link to the team, copy it and paste it here"

Write-Host "========================================================"

# Get the channel URL from user input
$channelUrl = Read-Host "Enter the Channel Link: Go to the channel, click on the three dots ..., click on Get a link to the channel, copy it and paste it here"

Write-Host "========================================================"

# Replace with tag name
$displayName = Read-Host "Enter the tag name"

Write-Host "========================================================"

# Extract the teamId from the URL
$teamId = Get-TeamIdFromUrl -url $teamUrl

# Extract the channelId from the URL
$channelId = Get-ChannelIdFromUrl -url $channelUrl

if ($null -ne $teamId -and $null -ne $channelId) {
    # Get an access token
    $body = @{
        grant_type    = "client_credentials"
        scope         = "https://graph.microsoft.com/.default"
        client_id     = $clientId
        client_secret = $clientSecret
    }

    try {
        $response = Invoke-RestMethod -Method Post -Uri "https://login.microsoftonline.com/$tenantId/oauth2/v2.0/token" -ContentType "application/x-www-form-urlencoded" -Body $body
        $accessToken = $response.access_token
    } catch {
        Write-Host "Error obtaining access token: $_"
        exit
    }

    # Define the API URLs
    $listTagsUrl = "https://graph.microsoft.com/v1.0/teams/$teamId/tags"
    $addMemberUrl = "https://graph.microsoft.com/v1.0/teams/$teamId/channels/$channelId/members"
    $listChannelMembersUrl = "https://graph.microsoft.com/v1.0/teams/$teamId/channels/$channelId/members"

    # Function to get tag ID by display name
    function Get-TagIdByDisplayName {
        param (
            [ValidateNotNullOrEmpty()]
            [string]$url,
            [ValidateNotNullOrEmpty()]
            [string]$token,
            [ValidateNotNullOrEmpty()]
            [string]$displayName
        )
        $headers = @{
            "Authorization" = "Bearer $token"
        }
        try {
            $response = Invoke-RestMethod -Uri $url -Method Get -Headers $headers
            $tag = $response.value | Where-Object { $_.displayName -eq $displayName }
            if ($tag) {
                return $tag.id
            } else {
                Write-Host "Tag with display name $displayName not found."
                return $null
            }
        } catch {
            Write-Host "Error getting tag ID: $_"
            return $null
        }
    }

    # Function to get members by tag
    function Get-MembersByTag {
        param (
            [ValidateNotNullOrEmpty()]
            [string]$url,
            [ValidateNotNullOrEmpty()]
            [string]$token
        )
        $headers = @{
            "Authorization" = "Bearer $token"
        }
        try {
            $response = Invoke-RestMethod -Uri $url -Method Get -Headers $headers
            return $response.value
        } catch {
            Write-Host "Error getting members by tag: $_"
            return $null
        }
    }

    # Function to get channel members
    function Get-ChannelMembers {
        param (
            [ValidateNotNullOrEmpty()]
            [string]$url,
            [ValidateNotNullOrEmpty()]
            [string]$token
        )
        $headers = @{
            "Authorization" = "Bearer $token"
        }
        try {
            $response = Invoke-RestMethod -Uri $url -Method Get -Headers $headers
            return $response.value
        } catch {
            Write-Host "Error getting channel members: $_"
            return $null
        }
    }

    # Function to add a member to a channel with retry mechanism
    function Add-MemberToChannel {
        param (
            [ValidateNotNullOrEmpty()]
            [string]$url,
            [ValidateNotNullOrEmpty()]
            [string]$token,
            [ValidateNotNullOrEmpty()]
            [string]$userId,
            [int]$retryCount = 5
        )
        $headers = @{
            "Authorization" = "Bearer $token"
            "Content-Type"  = "application/json"
        }
        $body = @{
            "@odata.type" = "#microsoft.graph.aadUserConversationMember"
            "roles"       = @()  # Add any roles if needed
            "user@odata.bind" = "https://graph.microsoft.com/v1.0/users/$userId"
        } | ConvertTo-Json

        for ($i = 0; $i -lt $retryCount; $i++) {
            try {
                $response = Invoke-RestMethod -Uri $url -Method Post -Headers $headers -Body $body
                return $response
            }
            catch {
                if ($_.Exception -match "429" -or $_.Exception -match "Too Many Requests") {
                    $delay = [Math]::Pow(2, $i)
                    Write-Host "Rate limit hit. Waiting for $delay seconds before retrying..."
                    Start-Sleep -Seconds $delay
                }
                else {
                    Write-Host "Error adding user $userId to the channel: $_"
                    return $null
                }
            }
        }
        Write-Host "Failed to add user $userId after $retryCount retries."
        return $null
    }

    # Function to process members in batches
    function Add-MembersInBatches {
        param (
            [ValidateNotNullOrEmpty()]
            [array]$members,
            [int]$batchSize,
            [int]$batchDelay,
            [ValidateNotNullOrEmpty()]
            [string]$token,
            [ValidateNotNullOrEmpty()]
            [array]$existingMembers
        )
        $totalMembers = $members.Count
        for ($i = 0; $i -lt $totalMembers; $i += $batchSize) {
            $batch = $members[$i..[Math]::Min($i + $batchSize - 1, $totalMembers - 1)]

            foreach ($member in $batch) {
                $userId = $member.userId
                $displayName = $member.displayName
                $isMember = $existingMembers | Where-Object { $_.userId -eq $userId }

                if (-not $isMember) {
                    $result = Add-MemberToChannel -url $addMemberUrl -token $token -userId $userId
                    if ($result) {
                        Write-Host "Added user $displayName to the channel."
                    } else {
                        Write-Host "Failed to add user $displayName to the channel."
                    }
                } else {
                    Write-Host "User $displayName is already a member of the channel."
                }
            }
            Write-Host "Processed batch of size $batchSize. Waiting for $batchDelay seconds before processing the next batch..."
            Start-Sleep -Seconds $batchDelay
        }
    }

    # Get the tag ID by display name
    $tagId = Get-TagIdByDisplayName -url $listTagsUrl -token $accessToken -displayName $displayName

    if ($null -ne $tagId) {
        # Get members by tag
        $listMembersUrl = "https://graph.microsoft.com/v1.0/teams/$teamId/tags/$tagId/members"
        $members = Get-MembersByTag -url $listMembersUrl -token $accessToken

        if ($null -ne $members) {
            # Get existing channel members
            $existingMembers = Get-ChannelMembers -url $listChannelMembersUrl -token $accessToken
            
            # Process members in batches
            Add-MembersInBatches -members $members -batchSize $batchSize -batchDelay $batchDelay -token $accessToken -existingMembers $existingMembers

            Write-Host "Finished processing all members."
        } else {
            Write-Host "No members found for the tag."
        }
    } else {
        Write-Host "Tag ID could not be retrieved. Please check the tag display name."
    }
} else {
    Write-Host "Team ID or Channel ID could not be retrieved. Please check the URLs."
}
```

## Config file

```json
{
    "tenantId": "<your-tenant-id>",
    "clientId": "<your-client-id>",
    "clientSecret": "<your-client-secret>",
    "batchSize": 30,
    "batchDelay": 60
}
```
## Dive into the script

This PowerShell script is used to add members to a Microsoft Teams channel based on a tag. It uses the Microsoft Graph API to interact with Microsoft Teams. Here's an overview of what the script does:

1. **Import-Config function:** This function loads a configuration from a JSON file. The configuration includes parameters like tenantId, clientId, clientSecret, batchSize, and batchDelay.
2. **Get-TeamIdFromUrl and Get-ChannelIdFromUrl functions:** These functions extract the teamId and channelId from the provided URLs.
3. **Get-TagIdByDisplayName function:** This function retrieves the ID of a tag based on its display name.
4. **Get-MembersByTag function:** This function retrieves the members associated with a specific tag.
5. **Get-ChannelMembers function:** This function retrieves the existing members of a channel.
6. **Add-MemberToChannel function:** This function adds a member to a channel. It includes a retry mechanism to handle rate limits from the Microsoft Graph API.
7. **Add-MembersInBatches function:** This function processes the addition of members in batches to avoid hitting rate limits. It checks if a user is already a member of the channel before trying to add them.
8. The script prompts the user for the team URL, channel URL, and tag name. It then extracts the teamId and channelId from the URLs.
9.  The script then retrieves the tag ID, members associated with the tag, and existing channel members. It adds the members to the channel in batches, skipping any members who are already in the channel.

## Dive into config file
The configuration file values are used for the following purposes:

1. **tenantId:** This is the ID of your Microsoft Entra tenant. It's used to authenticate your script with Microsoft Graph API.

2. **clientId and clientSecret:** These are the ID and secret of your Microsoft Entra ID app registration. They are used to authenticate your script with Microsoft Graph API. The script uses these values to get an access token, which is required to make authenticated requests to the Microsoft Graph API.

3. **batchSize:** This is the number of members to add in each batch. The script adds members to the Teams channel in batches to avoid hitting rate limits of the Microsoft Graph API.

4. **batchDelay:** This is the delay (in seconds) between each batch. It's used to prevent the script from hitting rate limits of the Microsoft Graph API by pausing between each batch of members being added.

By storing these values in a configuration file, you can easily change them without modifying the script itself.

## Prepare your configuration file

Create a new configuration json file `config.json` in the same location of your script. Copy the config example above and paste it inside your newly created file.

To get tenantId, clientId and clientSecret values follow the following steps:

1. Navigate to [portal.azure.com](https://portal.azure.com/)
2. Go to Microsoft Entra ID (Known before as Azure Active Directory Azure AD)
    ![Entra ID](/assets/img/add-to-teams-channel-by-tag/1-micorsoft-intra.png)
3. Click on App registrations then New registration
    ![New Registration](/assets/img/add-to-teams-channel-by-tag/2-new-registeration.png)
4. Add name to your app, then Register the app
    ![App Name](/assets/img/add-to-teams-channel-by-tag/3-app-name.png)
5. The new app created successfully copy the values of tenantId, and clientId and paste them inside the config file
    ![Client Tenant](/assets/img/add-to-teams-channel-by-tag/4-app-client-tenant-ids.png)
6. Create a new client secret
   - Click on Certificates & secrets
   - New client secret
   - Add Description to your secret
   - If you would like to change expires date
   - Then click on Add
   - Copy the value of the created secret and paste it inside the config file
    ![Client Secret](/assets/img/add-to-teams-channel-by-tag/5-new-secret.png)
    ![Secret value](/assets/img/add-to-teams-channel-by-tag/6-secret-value.png)
7. Add the required API permissions to the created App registration
   - Click on API permissions
   - Add permission
   - Select Microsoft Graph API
   - Select Application permissions
   - Then search for the first permission (ChannelMember.ReadWrite.All)
   - Select the permission
   - Click the "Add permissions" button
    ![Add Permission (1)](/assets/img/add-to-teams-channel-by-tag/7-add-permission.png)
    ![Add Permission (2)](/assets/img/add-to-teams-channel-by-tag/8-add-permission-2.png)
8. Repeat step (7) to add all the required permission. Here is the list of them
   - ChannelMember.ReadWrite.All
   - Group.Read.All
   - Group.ReadWrite.All
   - TeamMember.Read.All
   - TeamworkTag.Read.All
   - User.Read.All
9. Grant Admin Consent: Giving approval for the application to use added permissions
    - Click the "Grant admin consent" button
    - Click the "Yes" button for confirmation
    ![Grant Admin Consent](/assets/img/add-to-teams-channel-by-tag/9-grant-access.png)
10. After Granting Access, here is the list of the configured permissions:  
    ![Configured Permissions](/assets/img/add-to-teams-channel-by-tag/10-permissions-added.png)

>If you would like to learn more about Microsoft Graph permissions, check out the following link:
[Overview of Microsoft Graph permissions](https://learn.microsoft.com/en-us/graph/permissions-overview?tabs=http)
{: .prompt-info }

Your configuration file now is ready, and you are ready to run the script.

## Running the script

1. Create a new powershell file, and paste the script inside it
2. Open PowerShell: Start PowerShell from your Start menu or taskbar.
3. Navigate to Your Script: `cd path\to\your\script`
4. Execute the Script: `.\YourScriptName.ps1`
5. The script will prompt you for the Team link, channel link, and the tag name.
6. To get the team link, navigate to the desired MS teams team and click on the three dots ..., then get link to  team 
![Get team link](/assets/img/add-to-teams-channel-by-tag/11-get-team-link.png)
1. To get the channel link, navigate to the desired MS teams channel that you would like to add users to and click on the three dots ..., then get link to channel
![Get channel link](/assets/img/add-to-teams-channel-by-tag/12-get-channel-link.png) 
