---
layout: post
title:  "Automate Adding Users to a Security Group in Azure DevOps"
date:   2023-09-13 07:35:05 +0100
---

`azure-devops` `security-groups` `powershell-script`

This PowerShell script is designed to automate the process of adding users to a security group in Azure DevOps. 

Users are added to the specified group in batches to efficiently handle a large number of users. 

The script allows for customization by specifying the PAT, organization, project, group name, and batch size to meet your specific requirements.

{% highlight ruby %}
# Define your variables
$PAT = "YOUR_PERSONAL_ACCESS_TOKEN"
$Organization = "https://dev.azure.com/YOUR_ORGANIZATION_NAME/"
$Project = "YOUR_PROJECT_NAME"
$GroupName = "YOUR_GROUP_NAME"
$BatchSize = 100

# Log in with your Personal Access Token
echo $PAT | az devops login --org $Organization

# Set the default organization
az devops configure --defaults organization=$Organization

# Fetch group information
$group = az devops security group list --org $Organization --project $Project | 
         ConvertFrom-Json | 
         Select-Object -ExpandProperty graphGroups | 
         Where-Object { $_.displayName -eq $GroupName }

$groupId = $group.descriptor

# Initialize iteration counter
$iteration = 1

# Loop to fetch users in batches
$skip = 0
do {
    # Fetch users in the current batch
    $allUsers = az devops user list --org $Organization --top $BatchSize --skip $skip | 
                ConvertFrom-Json

    foreach ($user in $allUsers.members) {
        $userPrincipalName = $user.user.principalName
        
        # Add the user to the specified group
        az devops security group membership add --org $Organization --group-id $groupId --member-id $userPrincipalName
        
        Write-Host "Iteration $iteration - Added user '$userPrincipalName' to '$GroupName' group."
        $iteration++
    }

    $skip += $BatchSize
} while ($allUsers.members.Count -eq $BatchSize)
{% endhighlight %}

### Let's dive into the script to understand how it operates:

1. **Set Variables:** The script starts by setting the value of several variables:

    - $PAT: Personal Access Token to connect on Azure DevOps.
    - $Organization: Organization URL to list all users.
    - $Project: The name of the project.
    - $GroupName: The name of the security group to which users will be added.

2. **Log In:** The script uses the provided Personal Access Token (PAT) to log in to the Azure DevOps organization using the az devops login command. This allows the script to authenticate and perform actions on behalf of the user.

3. **Set Defaults:** The script sets the default organization for the Azure DevOps CLI using the `az devops configure` command. This means that subsequent commands won't need to specify the organization explicitly.

4. **Fetch Group Information:** The script uses the `az devops security group list` command to retrieve a list of security groups from the specified project within the organization. Then converts the output from JSON to a PowerShell object, selects the `graphGroups` property, and then filters the results to find a specific security group with a display name matching the value stored in the $GroupName variable.

5. **Add Users to Group:** The script then enters a loop where it fetches users in batches of 100 using the `az devops user list` command. It iterates through the list of users, retrieving their principal name (user identifier) and using the `az devops security group membership add` command to add each user to the previously identified security group.

6. **Output and Iteration:** Within the loop, the script outputs information about the iteration number and the user being added to the group using the `Write-Host` command.

7. **Looping:** The script continues looping and fetching users in batches until the total number of users fetched is less than the batch size (100).

**🔴IMPORTANT❗🔴**

The default limit for displaying project security group members in Azure DevOps is **500**,  So, If you plan to use the provided script to add more than 500 users, remember that only the first 500 will be visible. 

**References**

[az devops security group list](https://learn.microsoft.com/en-us/cli/azure/devops/security/group?view=azure-cli-latest#az-devops-security-group-list)

[az devops security group membership add](https://learn.microsoft.com/en-us/cli/azure/devops/security/group/membership?view=azure-cli-latest#az-devops-security-group-membership-add)

[az devops user list](https://learn.microsoft.com/en-us/cli/azure/devops/user?view=azure-cli-latest#az-devops-user-list)