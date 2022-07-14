# Azure Management Group

## Permissions for creating new management groups (hierarchy protection)

From the azure portal:

By default, all Azure Active Directory security principals can create new management groups. When this setting is turned on, security principals must have management group write access to create new management groups.

From the azure documentation:

Any Azure AD user in the tenant can create a management group without the management group write permission assigned to that user if hierarchy protection isn't enabled. This new management group becomes a child of the Root Management Group or the default management group and the creator is given an "Owner" role assignment. Management group service allows this ability so that role assignments aren't needed at the root level. No users have access to the Root Management Group when it's created. To avoid the hurdle of finding the Azure AD Global Admins to start using management groups, we allow the creation of the initial management groups at the root level.
(source: https://docs.microsoft.com/en-us/azure/governance/management-groups/create-management-group-azure-cli#prerequisites)

Setting - Require authorization

Any user, by default, can create new management groups within a tenant. Admins of a tenant may wish to only provide these permissions to specific users to maintain consistency and conformity in the management group hierarchy. If enabled, a user requires the Microsoft.Management/managementGroups/write operation on the root management group to create new child management groups.

source: https://docs.microsoft.com/en-us/azure/governance/management-groups/how-to/protect-resource-hierarchy#setting---require-authorization

### Create MG with AAD User

~~~ bash
# Assign Contributor Role to aad user peter
# We are logged in with global admin
userpeter=$(az ad user list --display-name peter --query [].id -o tsv)
subscope1=$(az account subscription list --query [0].id -o tsv)
subscope2=$(az account subscription list --query [1].id -o tsv)
az role assignment create --assignee $userpeter --role Contributor --scope $subscope1
az role assignment list --assignee $userpeter --scope $subscope1
az role assignment list --assignee $userpeter --scope $subscope2
# login with peter
az logout # logout global admin
az login # make sure you login with peter
az account show # verify you are logged in with peter
az account management-group list # not authorized to perform action on scope 'Microsoft.Management/managementGroups/read'
az account management-group create --name peter --display-name peter.parker # works
az account management-group delete --name peter # works
~~~

It looks like it is not enough to be contributor on a certain subscription to have the right to list management groups.
Let us assign a Management Group role to another aad user.

Assign the needed role to our global admin.

~~~ bash
userga=$(az ad user list --display-name ga --query [0].id -o tsv)
tenantid=$(az account show --query tenantId -o tsv)
mgscope=/providers/Microsoft.Management/managementGroups/$tenantid #the tenant root group has the same id like the tenant id.
az role assignment create --assignee $userga --role "Management Group Contributor" --scope $mgscope
az role assignment list --assignee $userga --scope $mgscope # the scope is not subscripton
az account management-group list # Works
~~~

We tried this also with another user called ga2 which only has the role "Management Group Contributor" assigned and did receive the following error message:

> (AuthorizationFailed) The client 'ga2@myedge.org' with object id 'xxxxxx-xxxx-xxxx-xxxx' does not have authorization to perform action 'Microsoft.Management/register/action' over scope '/subscriptions/xxxxxx-xxxx-xxxx-xxxx' or the scope is invalid. If access was recently granted, please refresh your credentials.
Code: AuthorizationFailed
Message: The client 'ga2@myedge.org' with object id 'xxxxxx-xxxx-xxxx-xxxx' does not have authorization to perform action 'Microsoft.Management/register/action' over scope '/subscriptions/xxxxxx-xxxx-xxxx-xxxx' or the scope is invalid. If access was recently granted, please refresh your credentials.

We need to assign the action 'Microsoft.Management/register/action'. We will try to find a role which does include the action and assign it to the user ga2.

~~~ bash
az logout
az login # login as global admin
az role definition list > roledeflist.txt
grep Microsoft.Management/register roledeflist.txt # nothing found :(
grep action roledeflist.txt # nothing found :(
grep -B5 -A20 "\"\*\"" roledeflist.txt # Only two roles which include all action and do not exclude our needed action owner, contributor
userid2=$(az ad user list --display-name ga2 --query [].id -o tsv)
tenantid=$(az account show --query tenantId -o tsv)
subscope2=$(az account subscription list --query [1].id -o tsv)
az role assignment create --assignee $userid2 --role Contributor --scope $subscope2
az role assignment list --assignee $userid2 --scope $subscope2
az logout # logout global admin
az login # login ga2
az account management-group list # works :)
az account management-group show -n $tenantid # works :)
~~~

### Create MG with AAD Service Principal

Test if we can create MG via AAD SP without any further rights.

Based on: 
- https://docs.microsoft.com/en-us/cli/azure/authenticate-azure-cli#sign-in-with-a-service-principal
- https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli

~~~ bash
az logout 
az login # login with gloabl admin
spn=sp-mg-test
subid=$(az account show --query id -o tsv)
tenantid=$(az account show --query tenantId -o tsv)
pw=$(az ad sp create-for-rbac --name $spn --query password -o tsv)
appid=$(az ad sp list --display-name $spn --query [0].appId -o tsv)
objectid=$(az ad sp list --display-name $spn --query [0].id -o tsv)
subscope2=$(az account subscription list --query [1].id -o tsv)
mgscope=/providers/Microsoft.Management/managementGroups/$tenantid #the tenant root group has the same id like the tenant id.
az role assignment create --assignee $objectid --role Contributor --scope $subscope2
az role assignment create --assignee $objectid --role "Management Group Contributor" --scope $mgscope
az login --service-principal --username $appid --password $pw --tenant $tenantid # login with sp
az account management-group list # works :)
az logout
az login # with global admin
az ad sp delete --id $appid
~~~


# Misc

git

~~~ bash
prefix=cptdmg
git config --global init.defaultBranch main
git init
gh repo create $prefix --public
git remote add origin https://github.com/cpinotossi/${prefix}.git
git add *
git commit -m"init"
git push origin main
~~~