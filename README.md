# An infrastructure in a box

#### *DISCLAIMER: This is NOT a production ready installation. There are many configration changes required for production readiness. DO NOT USE AS IS IN PRODUCTION, OR DO SO AT YOUR OWN RISK. MANY CHANGES ARE NOTED HERE BUT THIS NOT CAPTURE ALL OF THE REQUIRED CHANGES. This is a work in progress. You are warned.*

A single docker compose file to build an entire cloud for any business including
* [Basic Infrastructure](#Basic-Infrastructure)
* [Source Code Management and Analysis](#source-code-management-and-analysis)
* [Identity Access Management](#identity-access-management)
* [Cloud Application Runtimes](#cloud-application-runtimes)
* [Business Applications](#business-applications)

##### Definitions
 - `$Repo_root` when used below where you cloned the repository to. Unless you renamed it when you cloned it, it'll be `<folder you cloned into>/business-in-a-box`

### General Set Up
- Install Docker.
- Create file locations

    Several locations will need to be created so the various images' configurations and logs are accessible and modifyable outside the image. Some of these may already exist. 

    - Sonarqube:

    | Host                                       | Container           |
    |--------------------------------------------|---------------------|
    | $Repo_root/volumes/sonarqube/config        | /opt/sonarqube/conf |
    | $Repo_root/volumes/sonarqube/logs          | /opt/sonarqube/log  |
    | $Repo_root/volumes/extensions/sonarqube    | /opt/sonarqube/conf |
    | $Repo_root/volumes/sonarqube/data          | /opt/sonarqube/log  |
    |                                            |                     |

    - Syncope:

    | Host                                       | Container           |
    |--------------------------------------------|---------------------|
    | $Repo_root/volumes/syncope/logs/core       | /opt/syncope/log    |
    | $Repo_root/volumes/syncope/config/core     | /opt/syncope/conf   |
    | $Repo_root/volumes/syncope/logs/console    | /opt/syncope/log    |
    | $Repo_root/volumes/syncope/config/console  | /opt/syncope/conf   |
    | $Repo_root/volumes/syncope/logs/wa         | /opt/syncope/log    |
    | $Repo_root/volumes/syncope/config/wa       | /opt/syncope/conf   |
    | $Repo_root/volumes/syncope/logs/sra        | /opt/syncope/log    |
    | $Repo_root/volumes/syncope/config/sra      | /opt/syncope/conf   |
    | $Repo_root/volumes/syncope/logs/enduser    | /opt/syncope/log    |
    | $Repo_root/volumes/syncope/config/enduser  | /opt/syncope/conf   |

- Once those directories are in place, you can `docker-compose up` and wait for the the system to come online and get healthy (several services may restart during bootup).

- Once the system is healthy, you can begin configuring the following systems in order:
    - [Configuring Syncope](#configuring-syncope)
    - [Configuring Sonarqube](#configuring-sonarqube)
    - [More to come...](#work-in-progress)
  

### Basic Infrastructure:
#### NginX
Nginx is a reverse proxy. It'll let the outside world communicate with the inside of the docker container transparently. 

Because the Docker containers and Docker host end up using different URLs, we have to set up a reverse proxy. But don't worry, its in docker -- all that means is that the browser and the backend apps *mostly* won't know the difference.

The proxy _should have_, for security purposes: *TODO: need to set these up still - seems we can function in a development environment with out it.*
- An SSL certificate to deliver to clients.
- An SSL certificate to use between the services and the proxy.

But the proxy _needs_ in order to function:
- the proxy rules to map ouside docker -> inside docker

These rules are defined in [Nginx's configuration volume](./volumes/config/nginx). 
I will not go into how to configure [Nginx's rules](http://nginx.org/en/docs/), the provided rules should work out of the box in a development enviornment. 

### Source Code Management and Analysis
The following tools are useful for source code development, management, analysis, and deployment.

#### [Gitlab](https://gitlab.com/gitlab-org/gitlab)
A full featured source code repository with OAuth capabilities, and even CICD. 

#### [SonarQube](https://github.com/SonarSource/sonarqube)
A static code analysis tool -- helps find bugs and security issues quicker. 
When built the first time, the default admin username and password are both `admin` and you will be forced to change it on the first login. I changed it to password, but you must use something stronger especially in production and test environments. 

The following volumes are required to have access to the configuration, logs, and data that sonarqube uses. 
They map from the same locations on the docker host to the service's expected location.
| Host                                                 | Container           |
|------------------------------------------------------|---------------------|
| $Repo_root/volumes/sonarqube/config/**               | /opt/sonarqube/conf |
| $Repo_root/volumes/sonarqube/logs/**                 | /opt/sonarqube/log  |
| $Repo_root/volumes/sonarqube/extensions/**           | /opt/sonarqube/conf |
| $Repo_root/volumes/sonarqube/data/**                 | /opt/sonarqube/log  |

##### Configuring SonarQube
Sonarqube can use SAML to perform AuthNZ.  Syncope needs a [Service Provider (SP) Metadata file](sonarqube/volumes/syncope/config/wa/saml/sonarqube.0.xml) for Sonarqube, which itself needs a certificate if you want the added security of message signing (you do in prod). This file will be loaded into Syncope, and the data in it will also be added to Sonarqube. [Sonarqube's documentation](https://docs.sonarqube.org/latest/instance-administration/authentication/saml/overview/) on SAML is useful. The `sonar.auth.saml.sp.privateKey.secured`, and `sonar.auth.saml.sp.certificate.secured` define the Private Key and Certficate which can be generated [quite easily](https://www.baeldung.com/openssl-self-signed-cert#creating-a-self-signed-certificate) but do note that the key needs to be converted to PKCS8 format. This step is defined below as [Syncope's SAML Prequesite](#SAML-Prequesite) -- every SAML implementation requires this if they want to use message siging (again, you do).

We will now configure Sonarqube to use Syncope as the Identity Provider (IdP).
- Login to sonarqube as Admin, and navigate to Administration > Authentication > SAML:
- Set the Application Id to `http://localhost/wa/sp/sonarqube` -- this must match what is defined in Syncope.
- Set the Provider Name to `Syncope` -- this will display on the login page.
- Set the Provider Id to ` http://localhost/wa/idp` -- this in the Issuer provided by Syncope's IdP Metadata. 
- Set the SAML Login URL to `http://localhost/wa/idp/profile/SAML2/Redirect/SSO` -- this is also defined in Syncope's IdP Metadata.
- Set Identity Provider Certificate to the value from Syncope's IdP Metadata. 
- Set SAML user login attribute to `username`
- Set SAML user name attribute to `name`
- Set SAML user email attribute to `email`
- Set SAML group attribute to `groups`
- Ensure Sign Requests is _checked_.
- Set the Service provider private key to the PKCS8 key generated in the [Syncope + Sonarqube SAML Prereqs](#SAML-Prequesite).
- Set the Service provider certificate to the certificate generated in the [Syncope + Sonarqube SAML Prereqs](#SAML-Prequesite).

#### [Apache Archivia](https://github.com/apache/archiva)
An artifact repository - it stores and provides access to code artifacts such as executables, jars, etc. 

### Identity Access Management
#### [Apache Syncope](https://github.com/apache/syncope)
A full featured Identity Provider (IdP) and Identity Access Management (IAM) suite -- allowing OpenId Connect (OIDC), and Single-Sign On (SSO) for the applications deployed here. It includes several pieces: Console, Web Access (WA), Secured Remote Access (SRA), and End User. The Console is used to administor Users, Roles, and IAM generally. WA and SRA are both used to handle logins between systems each with a specific target. Enduser allows users to manage their identity as its known to the IdP. 

Some [very helpful and useful blogs about Apache Syncope](https://www.tirasa.net/en/blog/apache-syncope)

###### Do note that Syncope's WA is built on [Apache CAS](https://apereo.github.io/)

* The Console (Default credentials are `admin` and `passsword`)
* WA & SRA - the Access portal for SSO
* End User - An access portal where users can maintain their data in the system.
* Core - required to be running for things to work. 

Syncope needs the following volumes created for configuration and logs. Each part of Syncope gets it's own directory.  
| Host                                                 | Container         |
|------------------------------------------------------|-------------------|
| $Repo_root/volumes/syncope/logs/core      | /opt/syncope/log  |
| $Repo_root/volumes/syncope/config/core    | /opt/syncope/conf |
| $Repo_root/volumes/syncope/logs/console   | /opt/syncope/log  |
| $Repo_root/volumes/syncope/config/console | /opt/syncope/conf |
| $Repo_root/volumes/syncope/logs/wa        | /opt/syncope/log  |
| $Repo_root/volumes/syncope/config/wa      | /opt/syncope/conf |
| $Repo_root/volumes/syncope/logs/sra       | /opt/syncope/log  |
| $Repo_root/volumes/syncope/config/sra     | /opt/syncope/conf |
| $Repo_root/volumes/syncope/logs/enduser    | /opt/syncope/log  |
| $Repo_root/volumes/syncope/config/enduser | /opt/syncope/conf |


##### Syncope Set up 
The following volumes are required to have access to the configuration, logs, and data that syncope uses. 
They map from the same locations on the docker host to the service's expected location. I've found it's quite easy to copy over the default configuration in the container to the host so that Syncope can be easily configured. 

* $Repo_root/volumes/syncope/config/** -> /opt/syncope/conf
* $Repo_root/volumes/syncope/logs/** -> /opt/syncope/log

When running in docker, these are paths and ports the applications are accessible from.
* REST API  http://localhost/syncope/
* Admin UI http://localhost/console/ * Credentials: admin / password * 
* End-user UI http://localhost/user/
* WA http://localhost/wa/ 
* SRA http://localhost/sra

Once Syncope is set up you can configure Sonarqube. If you want to use OpenID Connect (OIDC) instead of SAML, you will need a plugin for SonarQube as it does not support OIDC natively. Syncope, fortunately, supports several protocols.

###### Configuring Syncope
Syncope is quite large and will take some time to learn. We need to configure a few things before we're ready to use it. 
We need to define a set of rules, a set of policies, an authentication module, and a set of application configurations so that Syncope knows who Sonarqube or any other application is. We will use SAML to make the connection and get the users info into the other applications.

Once docker is up and running, you may begin configuring Syncope. 

* Rules (Configuration > Implementations)
    - Account (Configuration > Implementations > Account_Rule)
        The Account rule defines some defaults for accounts within Syncope.
        - Click the Green plus button
        - In the grey popup to the left
            - Select `Java`
            - Click the button with the Green Icon 
        - In the pop up Form:
            - Set the Key to `Default Account Rule`
            - Set the Class to `org.apache.syncope.common.lib.policy.DefaultAccountRuleConf`
            - Selecting `DefaultAccountRuleConf` will autofill the text area with JSON that looks like this:
            ```
            {"_class":"org.apache.syncope.common.lib.policy.DefaultAccountRuleConf","name":"org.apache.syncope.common.lib.policy.DefaultAccountRuleConf","maxLength":100,"minLength":3,"pattern":null,"allUpperCase":false,"allLowerCase":false,"wordsNotPermitted":[],"schemasNotPermitted":[],"prefixesNotPermitted":[],"suffixesNotPermitted":[]}
            ```
            - We will see this later, for now do not edit it. 
            - Click Save and be glad we have rules for accounts.
    - Password (Configuration > Implementations > Password_Rule)
        The Password rule defines, as you've probably guessed, rules for passwords.
        - Click the green plus button
        - In the grey popup to the left
            - Select `Java`
            - Click the button with the Green Icon 
        - In the pop up Form:
            - Set the Key to Key to `Default Password Rule`
            - Set the Class to `org.apache.syncope.common.lib.policy.DefaultPasswordRuleConf`, again autofilling the text area with:
            ```
            {"_class":"org.apache.syncope.common.lib.policy.DefaultPasswordRuleConf","name":"org.apache.syncope.common.lib.policy.DefaultPasswordRuleConf","maxLength":100,"minLength":3,"alphabetical":0,"uppercase":0,"lowercase":0,"digit":0,"special":0,"specialChars":[],"illegalChars":[],"repeatSame":0,"usernameAllowed":false,"wordsNotPermitted":[],"schemasNotPermitted":[]}
            ```
            - We will see this later, for now do not edit it. 
            - Click Save and now we have rules for passwords.

* Policies (Configuration > Policies)
    - Account (Configuration > Policies > Account)
        - Click the green + button to add a new Account Policy and:
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
            - Set the Configuration to `org.apache.syncope.common.lib.policy.DefaultAccessPolicyConf`. There are others to choose from, for now this suits the needs.
            - Click Save.
        - Click the `Default Access Policy` item in the list.
        - Click Configuration in the grey popup box on the right. On the next popup click the + button to add a new entry. Then:
            - Set the Order to 0 so that it's first.
            - Check Enabled, SSO Enabled, Require All Attributes.
            - Do not Check ` Case Insensitive`
            - Ignore Unauthorized Redirect URL for now.
        - Click Save.
    - Attribute Release
        - Click the Green `+` button to create a new Attribute Release Policy
        - In the popup menu
            - Set the Name to Default Attribute Release Policy
            - Set the Order to 0
            - Check the Attribute Consent checkbox to require Users to consent to those attributes - we'll see this in action later.
            - Click Save
        - Click the new Row and then Configuration in the grey popout menu to the right.
        - In the popup menu
            - syncopeUserAttr_email and syncopeUserAttr_username
    

    - Authnetication (Configuration > Policies > Authentication)
        - Click the + button to add a new Authentication Policy and:
            - Set the Name to `Syncope Authentication Policy`
            - Click Save
        - Click the `Syncope Authentication Policy` item in the list.
        - Click Configuration in the grey popup box on the right. Then on the next popup, Syncope should be in the Available list. Click the arrow point to the Selected list to tell Syncope to use it for this policy. 



* Authentication Module (WA > Authentication Modules)
This defines our "method of authentcation." More specifically it defines a way someone can authenticate to Syncope - in this case, we will use Syncope as the method but there are others available which require different settings.
    - Set the Key to Syncope Authentication
    - Ignore Description (unless you really want to add it)
    - Set the State to Active
    - Set the order to 0 
    - Set the Type to `org.apache.syncope.common.lib.auth.SyncopeAuthModuleConf`
    - Click "Next"
    - Set the Domain (which is also known as a Realm) to Master
    - Click "Next" 
    - Add the following mapping of attributes:
        | Internal attribute           | External attribute | Did we make? |
        |------------------------------|--------------------|--------------|
        | syncopeUserAttr_name         | name               | ✅ (Schema)  |
        | syncopeUserAttr_email        | email              | ❌           |
        | syncopeUserKey               | sub                | ❌           |
        | syncopeUserDynMemberships    | groups             | ❌           |
        | username                     | username           | ❌           |
    - Click "Finish"
* User (Realms > User)
    - 
* Group (Realms > Group)
    - 

* Sonarqube OIDC (WA > Client Applications > OIDCRP)
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

* Sonarqube SAML (WA > Client Applications > SAML2SP)
    #### SAML Prequesite
    We must genrerate a private key, a certificate, and a SAML Service Provider's metadata file. The certificate and key will be loaded into Sonarqube when we configure it's end of the SAML protocol. Syncope will also need the certificate. We'll also need to create a PKCS8 formatted version of the private key. These files must be securely transmitted and stored - they are _critical_ to the security of the SAML authentication process.
    Once the cert, and the 2 versions of the key are in place, you can [create a SAML metadata for the service provider](https://www.samltool.com/sp_metadata.php) *Beware: private keys added there are sent to their server. Trust at your own risk and do not use it for production keys.  
    
    ###### Protect these files like you protect your identity because that's literally what these files are - the identity of Sonarqube as far as Syncope is concerned. If these are leaked then the authentication process is no longer secure. In production store them in a secrets manager. 
    ```
    $ openssl req -x509 -newkey rsa:4096 -keyout sonarqube.key -out sonarqube.crt -days 365 # generate a new key and cert that'll last for a year.
    $ openssl pkcs8 -topk8 -in sonarqube.key -out sonarqube.pkcs8.key -nocrypt # create PKCS8 version of the private key
    ```
    [Example Metadata file](./volumes/syncope/config/wa/saml/sonarqube.0.xml) This file is the file used by Syncope to validate the data Sonarqube sends it when doing a SAML exchange. All Service Provider metadata files will look similar with different values for various field such as `EntityId` and `validUntil`. 
    ```
        Double check your validUntil field -- it must be a future date.
    ```
    - Have the Prerequistes above in place. 
    - Click the Green Plus Icon and in the Popup menu:
        - Set the Realm to `\`.
        - Set the Name to `sonarqube`.
        - Set the ID to 1.
        - Set the Description to nothing or whatever you want.
        - Set the Access Policy to the one we defined earlier `Default Access Policy`.
        - Set the Attribute Release Policy to the one we defined earlier `Default Attribute Release Policy`.
        - Set the Authentication Policy to the one we defined earlier `Default Authentication Policy`.
        - Leave the Theme blank.
        - Set the EntityId to `http://localhost/wa/sp/sonarqube` or whatever you've defined in Sonarqube's metadata.
        - Set the Metadata Location to `file:///opt/syncope/conf/saml/sonarqube.0.xml` -- this path is a docker container path, linked in via a volume. What is in the repository should be good enough to get things started, but you'll have tpo update it for production use. 
        - Leave the Metadata Signature Location blank -- this will need to be set for production.
        - Ensure Sign Assertions is unchecked.
        - Ensure Sign Responses is _checked_.
        - Ensure Encryption Optional is unchecked.
        - Ensure Encrypt Assertions is unchecked.
        - Leave the Authentication Context Class blank.
        - Set NameId Format to `EMAIL_ADDRESS`
        - Leave Skew Allowance blank. 
        - Leave NameId Qualifier blank.
        - Leave Assertion Audiences blank.
        - Leave Service Provider NameId Qualifier blank. 
        - And finally, for now, leave the Algorithms as is so none are Selected. This will need to change for a production environment - you'll want only the strongest of them for production. 


To harden your Syncope system and get it ready for production:
* Note the changes mentioned else where for production, and change those.
* Change the default admin password and force its rotation every month; or disable the user.
* Remove Unused, weak, or lowbit algorthyms for the OIDC, OAUTH2, SAML, and other auth protocols. 
* Modify the logging configuration to not emit tokens.



#### Cloud Application Runtimes
##### [Apache ServiceComb](https://servicecomb.apache.org/)
A microservice tool - it lets you set up and run microservices.

##### [Apache OpenWhisk](https://github.com/apache/openwhisk)
A serverless function platform. 

##### [Apache Karaf](https://github.com/apache/karaf)
An full featured application runtime playing a similar role as ServiceComb

##### [Apache Skywalking](https://github.com/apache/skywalking)
An Application Performance Monitoring tool. Allows devs, leaders, and the whole team see how each of your Applications are performing. 

##### [Apache Kafka](https://github.com/apache/kafka)
A Pub-Sub message broker. Allows injestion of notifications to and from systems. 





#### Business Applications
The following applications will be deployed to and run in the cloud services set up above. The provide some core business, mission critical, functionality including communication and E-Commerce.
##### [Apache Finrect](https://github.com/apache/fineract)
A reliable, robust, and affordable core banking solution for financial institutions -- be your own bank!
##### [Apache OfBiz](https://github.com/apache/ofbiz-framework)
Framework components and business applications for ERP, CRM, E-Business/E-Commerce, Supply Chain Management and Manufacturing Resource Planning.
##### [Apache Roller](https://github.com/apache/roller)
A blog application. 
##### [Apache James](https://github.com/apache/james-project)
An Email server
##### [Apache OpenMeetings](https://github.com/apache/openmeetings)
Think Webex or Zoom -- but yours and free.





#### Honorable Mentions
##### [Apache Cloudstack](https://github.com/apache/cloudstack)
Cloudstack is a pretty interesting, well known, and highly used cloud solution. However, running it in Docker seems counter-intuitive as it's better when on bare metal.
##### [Apache Alura](https://github.com/apache/allura)
Alura is a neat software forge but many of the features it has is also offered by gitlab - and many that aren't. It may still make it in...



##### Work In Progress
This is a work in progress that is not yet, and may never be especially without proper configuration, ready for production deployments. Many of the applications listed here may or may not make it into the final Box. But there is a vision and a plan; and this is still a work in progress.

The plan is to include Gitlab, Archivia, and one of ServiceComb or OpenWhisk; followed by the business applications; and then the rest of Cloud Application Runtimes for future applications. 