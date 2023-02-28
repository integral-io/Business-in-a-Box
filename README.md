# An infrastructure in a box

*This is not a productiob ready installation. There are many configration changes required for production readiness.*

Several docker compose files to build up an entire cvloud infrastructure including
* Source Code Management and Analysis
* Identity Access Management
* Cloud Application Runtimes
* Business Applications
### Set Up
Several volumes will need to be created so the various images' configurations and logs are accessible and modifyable outside the image. Once those are in place, you can `docker-compose up` and wait for the the system to come online and get healthy (several services may restart during bootup). 

Because the Docker containers and Docker host end up using different URLs, we have to set up a reverse proxy. But don't worry, its in docker -- all that means is that the browser and the backend apps won't know the difference. This also means more configuration for the containers like SSL certificates so the reverse proxy can securely communicate with the backends.


#### Source Code Management and Analysis
The following tools are useful for source code development, management, analysis, and deployment.

##### Gitlab
A full featured source code repository with OAuth capabilities, and even CICD. 

##### Sonarcube
A static code analysis tool -- helps find bugs and security issues quicker. 
When built the first time, the default admin username and password are both `admin` and you will be forced to change it on the first login. I changed it to sonar, but you should use something strionger - especially in a production environment. 

The following volumes are required to have access to the configuration, logs, and data that sonarqube uses. 
They map from the same locations on the docker host to the service's expected location.
* InfraInABox/sonarqube/volumes/config/sonarqube/** -> /opt/sonarqube/conf
* InfraInABox/sonarqube/volumes/logs/sonarqube/** -> /opt/sonarqube/log
* InfraInABox/sonarqube/volumes/extensions/sonarqube/** -> /opt/sonarqube/conf
* InfraInABox/sonarqube/volumes/data/sonarqube/** -> /opt/sonarqube/log


##### Apache Archivia *
An artifact repository

#### Identity Access Management
##### Apache Syncope
A full featured Identity Provider (IdP) and Identity Access Management (IAM) suite -- allowing OpenId Connect (OIDC), and Single-Sign On (SSO) for the applications deployed here. It includes several pieces: Console, Web Access (WA), Secured Remote Access (SRA), and End User. The Console is used to administor Users, Roles, and IAM generally. WA and SRA are both used to handle logins between systems each with a specific target. Enduser allows users to manage their identity as its known to the IdP. 

* The Console (Default credentials are `admin` and `passsword`)
* WA & SRA - the Access portal for SSO

###### Set up 

The following volumes are required to have access to the configuration, logs, and data that syncope uses. 
They map from the same locations on the docker host to the service's expected location. I've found it's quite easy to copy over the default configuration in the container to the host so that Syncope can be easily configured.  

* InfraInABox/sonarqube/volumes/config/syncope/** -> /opt/syncope/conf
* InfraInABox/sonarqube/volumes/logs/syncope/** -> /opt/syncope/log

When running in docker, these are paths and ports the applications are accessible from.
* REST API  http://localhost:18080/syncope/
* Admin UI http://localhost:28080/syncope-console/ * Credentials: admin / password * 
* End-user UI http://localhost:38080/syncope-enduser/
* WA http://localhost:48080/syncope-wa/ 
* SRA http://localhost:58080/

###### Configuring Syncope
Syncope is quite large and will take some time to learn. We need to configure a few things before we're ready to use it. 
We need to define a set of rules, a set of policies, and a set of Application configurations so that Syncope knows who sonarqube is. 
* Authentication Module (WA > Applications > OIDCRP)
This defines our "method of authentcation." More specifically it defines a way someone can authenticate to Syncope.
    - Set the Key to Syncope
    - Ignore Description (unless you really want to add it)
    - Set the order to 0 
    - Set the Type to `org.apache.syncope.common.lib.auth.SyncopeAuthModuleConf`
    - Set the Realm to \ 
    - Add an internal to external attribute mapping and set the internal attribute to `syncopeUserAttr_email` and set the external attribute to `email`. 
* Rules
    - Account (Configuration > Implementations > Account_Rule)
    The Account rule defines some defaults for accounts within Syncope.
        - Set the Key to `Default Account Rule`
        - Set the Class to `org.apache.syncope.common.lib.policy.DefaultAccountRuleConf`
        - Selecting `DefaultAccountRuleConf` will autofill the text area with JSON that looks like this:
        ```
        {"_class":"org.apache.syncope.common.lib.policy.DefaultAccountRuleConf","name":"org.apache.syncope.common.lib.policy.DefaultAccountRuleConf","maxLength":100,"minLength":3,"pattern":null,"allUpperCase":false,"allLowerCase":false,"wordsNotPermitted":[],"schemasNotPermitted":[],"prefixesNotPermitted":[],"suffixesNotPermitted":[]}
        ```
        - We will see this later, for now do not edit it. 
    - Password (Configuration > Implementations > Password_Rule)
        - Set the Key to Key to `Default Password Rule`
        - Set the Class to `org.apache.syncope.common.lib.policy.DefaultPasswordRuleConf`, again autofilling the text area with:
        ```
        {"_class":"org.apache.syncope.common.lib.policy.DefaultPasswordRuleConf","name":"org.apache.syncope.common.lib.policy.DefaultPasswordRuleConf","maxLength":100,"minLength":3,"alphabetical":0,"uppercase":0,"lowercase":0,"digit":0,"special":0,"specialChars":[],"illegalChars":[],"repeatSame":0,"usernameAllowed":false,"wordsNotPermitted":[],"schemasNotPermitted":[]}
        ```
        - We will see this later, for now do not edit it. 
* Policies (Configuration > Policies)
    - Account Password (Configuration > Policies > Account)
        - Click the + button to add a new Account Policy and:
            - Set the Name to `Default Account Policy`
            - Set the Max Authetication Attempts to something sensible like 3 or 5 - 7 if you're feeling risky.
            - Check Propagate Suspension
            - Click Save
        - Click the `Default Account Policy` item in the list.
        - Click Rules in the grey popup box on the right and on the next popup, click the + button to add a new entry. Then:
            - Select the Default Account Rule from before
            - On the next screen we will see that JSON from before in a webform
            - Set some sensible defaults - this is pretty finegrained control, so keep it simple until its ready for production!
    - Password (Configuration > Policies > Password)
        - Click the + button to add a new Password Policy and:
            - Set the Name to `Default Password Policy`
            - Set the History Length to something sensible like 3 or 5 - 7 if you're feeling risky.
            - Do not check Allow Null passwords -- unless you really want to and understand the secuity implications of a user with a null password.
            - Click Save
        - Click the `Default Password Policy` item in the list.
        - Click Rules in the grey popup box on the right and on the next popup, click the + button to add a new entry. Then:
            - Select the Default Password Rule from before
            - On the next screen we will see that JSON from before in a webform
            - Set some sensible defaults - this is pretty finegrained control, so keep it simple until its ready for production!
    - Access (Configuration > Policies > Access)
        - Click the + button to add a new Access Policy and:
            - Set name to `Default Access Policy`
            - Set the Configuration to `org.apache.syncope.common.lib.policy.DefaultAccessPolicyConf`
            - Click Save
        - Click the `Default Access Policy` item in the list.
        - Click Configuration in the grey popup box on the right. On the next popup click the + button to add a new entry. Then:
            - Set the Order to 0 so that it's first.
            - Check Enabled, SSO Enabled, Require All Attributes.
            - Do not Check ` Case Insensitive`
            - Ignore Unauthorized Redirect URL for now.
        - Click Save
    - Authnetication (Configuration > Policies > Authentication)
        - Click the + button to add a new Authentication Policy and:
            - Set the Name to `Syncope Authentication Policy`
            - Click Save
        - Click the `Syncope Authentication Policy` item in the list.
        - Click Configuration in the grey popup box on the right. Then on the next popup, Syncope should be in the Available list. Click the arrow point to the Selected list to tell Syncope to use it for this policy. 
    - Sonarqube OIDC (WA > Client Applications > OIDCRP)
        - Set the realm to `/`
        - Set the name to Sonarqube (or some other app name)
        - Set the ID to 1
        - Ignore Description (unless you really want to add it)
        - Set the Access to policy to the one created earlier.
        - Ignore the Attribute Release Policies
        - Set the Authentiocation Policy to the one created earlier.
        - Set the Client ID and Secret -- REMEMBER THESE VALUES, you'll need them later to set up the OIDC IDP in the other application -- in this case its Sonarqube and thats the  app that'll get these values, but you could very well have another application defined here and it must use its own values for these fields. This is how Syncope "knows" the application and allows AuthNZ to ocurr through it.
        - Check `Sign IdToken`, `JWT Access Token`, and `Bypass Approval Prompt`
        - Set the Subject Type to `PUBLIC`
        - Set the `redirect_uri` to `http://localhost:9000/oauth2/callback/oidc` if you're setting up sonarqube, or to the correct url if you're setting up a different application.
        - Add all Supported Grant Types and Supported Response Types
        - Ignore Logout Uri


To harden your syncope system and get it ready for production:
    * Change the default admin password and force its rotation every month; or disable the user.
    * Remove Unused algorthyms for the OIDC ans OAUTH2 configuration. 



#### Cloud Application Runtimes
##### Apache ServiceComb
A microservice tool - it lets you set up and run microservices.

###### Using it

##### Apache OpenWhisk *
A serverless function platform. 

##### Apache Karaf *
An full featured application runtime playing a similar role as ServiceComb


##### Apache Skywalking
An application Performance monitoring tool. Allows devs, leaders, and the whole team see how each of your applications are performing. 

##### Apache Kafka  *
A Pub-Sub message broker. Allows injestion of notifications to and from systems. 





#### Business Applications
The following applications will be deployed to and run in the cloud services set up above. The provide some core business, mission critical, functionality including communication,  
##### Apache Finrect *
https://github.com/apache/fineract
A banking app -- be your own bank!
##### Apache OfBiz *
A full featured set of services for the entire business.
##### Apache Roller *
A blog application. 
##### Apache James *
An Email server
##### Apache OpenMeetings *
Think Webex or Zoom -- but yours and free.





#### Honorable Mentions
##### Apache Cloudstack
##### Apache Alura