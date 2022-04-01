
# Prerequisites
This is the starting point for the end-to-end instructions on deploying the [AKS Baseline for Regulated Workloads reference implementation](/README.md). There is required access and tooling you'll need in order to accomplish this. Follow the instructions below and on the subsequent pages so that you can get your environment and subscription ready to proceed with the AKS cluster creation.

## Azure AD Tenant
Azure AD [User Administrator](https://docs.microsoft.com/azure/active-directory/users-groups-roles/directory-assign-admin-roles#user-administrator-permissions) is _required_ to create a "break glass" AKS admin Active Directory Security Group and User. For this exercise, consider [creating a new tenant](https://docs.microsoft.com/azure/active-directory/fundamentals/active-directory-access-create-new-tenant#create-a-new-tenant-for-your-organization) to use while evaluating this implementation. 
Alternatively, you could get your Azure AD admin to create this for you when instructed to do so.

