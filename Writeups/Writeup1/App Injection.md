
# Maintaining Persistance in Azure through Application Injection Using GraphRunner Tool
### Scenario Overview: 
Once an attacker compromises a user in an Azure environment, maintaining persistent access is a key goal to ensure continued control over the system. One effective method is to inject an application on behalf of the compromised user, leveraging tools like GraphRunner. This method allows attackers to interact with Azure resources and APIs without needing direct access to the compromised user's credentials continuously.

Before exploring the attack scenario, let's first cover the key concepts in Azure.

### Key concepts
<b>What is Persistance?</b>
<br />
Persistence in Azure is a post-exploitation technique that enables attackers to maintain long-term access to a compromised environment. By creating or modifying resources and configurations, attackers establish persistent access, ensuring they can return even if their original entry point is closed or detected. 

<b>What is App Registation in Azure?</b>
<br />
App Registration in Azure refers to the process of registering an application with Azure Active Directory (Azure AD) so that it can authenticate users, access resources, and interact with other services in Azure. When an app is registered, Azure AD creates a unique identity for it, which is used  to authenticate users and securely access resources in Azure, such as Microsoft Graph, storage accounts, or databases.

<b>What is GraphRunner Tool?</b>
<br />
GraphRunner is a post-exploitation toolset for interacting with the Microsoft Graph API. It provides various tools for performing reconnaissance, persistence, and pillaging of data from a Microsoft Entra ID (Azure AD) account.

### Attack Scenario
![image](https://github.com/ArjunAshok21/AzureRedteaming/blob/3034675f504c7a1b3de4942d87a0c85c166836e9/Post-Exploitation/Application%20Injection/Attack%20Path.png)
### Technical Summary:

Once you compromised a user in azure, By default, users can create applications. But, by default, they cannot add administrative privileges such as Directory.ReadWrite.All. They can, however, add a number of delegated privileges that do not require admin consent by default. Most of these privileges that do not require admin consent are for performing common tasks such as reading email (Mail.Read), listing users in the directory (User.ReadBasic.All), navigating SharePoint and OneDrive (Files.ReadWrite.All and Sites.ReadWrite.All), and many more.

So by abusing this feature,The user can deploying an app with specific permissions and consenting to it as a compromised user, we can then use the service principal credentials associated with the application to access the user’s account. Even if the compromised user changes their password, the app maintains access to their account. If all sessions are terminated for the compromised user, we still retain access through the app's access token, which allows continued operation as the user until the token expires.

### Exploitation

1) Authenticate GraphRunner using the compromised credentials, considering the compromised user is Testvictm1.

![image](https://github.com/ArjunAshok21/AzureRedteaming/blob/3034675f504c7a1b3de4942d87a0c85c166836e9/Post-Exploitation/Application%20Injection/Images/Picture1.png)


2) In GraphRunner, the Invoke-InjectOAuthApp module which automates the deployment of an app registration to a Microsoft tenant. If the Azure portal is restricted, this module provides an alternative method for app deployment, provided that users are allowed to register applications within the tenant. Here we are using -scope parameter to “op backdoor”, the tool will create an app and add a large number of common permissions to it, including access to Mail, Files, Teams, and more. None of these permissions require admin consent.

![image](https://github.com/ArjunAshok21/AzureRedteaming/blob/3034675f504c7a1b3de4942d87a0c85c166836e9/Post-Exploitation/Application%20Injection/Images/Picture2.png)

3) Once the app is deployed, a consent URL is automatically generated in the terminal. This URL is unique and associated with the specific app registration, including all the requested permissions. When a user visits this URL, they will be prompted to grant consent for the app’s requested permissions.So, here we are granting consent for the app on behalf of the compromised user.

![image](https://github.com/ArjunAshok21/AzureRedteaming/blob/3034675f504c7a1b3de4942d87a0c85c166836e9/Post-Exploitation/Application%20Injection/Images/Picture3.png)

![image](https://github.com/ArjunAshok21/AzureRedteaming/blob/3034675f504c7a1b3de4942d87a0c85c166836e9/Post-Exploitation/Application%20Injection/Images/Picture4.png)

4) When an application with delegated permissions is consented to, we need to catch the OAuth code that is sent to the specified redirect URI in order to complete the flow and obtain access tokens.Here we are performing persistence within the account we control, it’s possible to complete this flow by directing the browser to localhost.To perform the action we have a module named Invoke-AutoOAuthFlow which setup a  minimal web server to listen for this request and completes the OAuth flow with the provided app registration credentials. when we set the Reply URL such as “http://localhost:8000” in InjectOAuthApp.it will automatically detect it and ouput the exact command needed to run in another terminal to catch and complete the flow.

![image](https://github.com/ArjunAshok21/AzureRedteaming/blob/3034675f504c7a1b3de4942d87a0c85c166836e9/Post-Exploitation/Application%20Injection/Images/Picture6.png)


5) Before Accept the consent, Run Invoke-AutoOAuthFlow command and It will listen for requests to it containing the OAuth code and automatically complete the flow using the service principal’s credentials. Upon successfully completing the flow, it will output a new set of access tokens.

![image](https://github.com/ArjunAshok21/AzureRedteaming/blob/3034675f504c7a1b3de4942d87a0c85c166836e9/Post-Exploitation/Application%20Injection/Images/Picture5.png)

6) Once Successfully consented and injected the application, the application can run in the background, allowing the attacker to maintain persistent access.  The application is running under the context of the compromised user and uses Microsoft Graph API, it’s harder to detect compared to traditional methods like direct login attempts. This allows the attacker to blend in with normal operations.

