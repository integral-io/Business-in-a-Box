# An infrastructure in a box

#### *DISCLAIMER: This is NOT a production ready installation. There are many configuration changes required for production readiness. DO NOT USE AS IS IN PRODUCTION, OR DO SO AT YOUR OWN RISK. MANY CHANGES ARE NOTED HERE BUT THIS DOES NOT CAPTURE ALL OF THE REQUIRED CHANGES. This is a work in progress. You are warned.*

A single docker compose file to build an entire cloud for any business including
* [Basic Infrastructure](#Basic-Infrastructure)
* [Source Code Management and Analysis](#source-code-management-and-analysis)
* [Identity Access Management](#identity-access-management)
* [Cloud Application Frameworks](#cloud-application-frameworks)
* [Business Applications](#business-applications)

##### Definitions
###### Define these as environment variables, or use the [.env](./.env) file
 - `$Repo_root` when used below where you cloned the repository to. Unless you renamed it when you cloned it, it'll be `<folder you cloned into>/business-in-a-box`
 - `$Sonarqube_HOME` is the volume directory for SonarQube's configs and logs (`$Repo_ROOT/volumes/sonarqube`)
 - `$Syncope_HOME` is the volume directory for Syncopes's configs and logs (`$Repo_ROOT/volumes/syncope`)
 - `$Nginx_HOME` is the volume for Nginx's configs and logs (`$Repo_ROOT/volumes/nginx`)

### General Set Up
- Install Docker.
- Create file locations
    Several locations will need to be created so the various images' configurations and logs are accessible and modifiable outside the image. Some of these may already exist. There are also several named volumes that do not need to be created. *Some map to the same locations in the containers. This is normal. I've found it's quite easy to copy over the default configuration from the container to the host so that Syncope can be easily configured if you accidentally lose your config, or want to start from scratch. This can be achieved by temporarily targeting these directories in the container, then accessing the containers terminal via Docker, and copying all files into the new target on the container -- this will copy it out of the host and into the volume. Then return the container's target to the original value and the service will pick up the new config from the volume.*
    - Nginx (here we "volume" a file)

    | Host                           | Container                 |
    |--------------------------------|---------------------------|
    | $Nginx_HOME/config/nginx.conf  | /etc/nginx/nginx.conf     |

    - SonarQube:

    | Host                          | Container                 |
    |-------------------------------|---------------------------|
    | $Sonarqube_HOME/config        | /opt/sonarqube/conf       |
    | $Sonarqube_HOME/logs          | /opt/sonarqube/log        |
    | $Sonarqube_HOME/extensions    | /opt/sonarqube/extensions |
    | $Sonarqube_HOME/data          | /opt/sonarqube/log        |

    - Syncope Core:

    | Host                          | Container           |
    |-------------------------------|---------------------|
    | $Syncope_HOME/logs/core       | /opt/syncope/log    |
    | $Syncope_HOME/config/core     | /opt/syncope/conf   |

    - Syncope Console:

    | Host                          | Container           |
    |-------------------------------|---------------------|
    | $Syncope_HOME/logs/console    | /opt/syncope/log    |
    | $Syncope_HOME/config/console  | /opt/syncope/conf   |

    - Syncope WA: 

    | Host                          | Container           |
    |-------------------------------|---------------------|
    | $Syncope_HOME/logs/wa         | /opt/syncope/log    |
    | $Syncope_HOME/config/wa       | /opt/syncope/conf   |

    - Syncope Sra:

    | Host                          | Container           |
    |-------------------------------|---------------------|
    | $Syncope_HOME/logs/sra        | /opt/syncope/log    |
    | $Syncope_HOME/config/sra      | /opt/syncope/conf   |

    - Syncope End User:

    | Host                          | Container           |
    |-------------------------------|---------------------|
    | $Syncope_HOME/logs/enduser    | /opt/syncope/log    |
    | $Syncope_HOME/config/enduser  | /opt/syncope/conf   |

- Once those directories are in place, you can `docker compose up` and wait for the the system to come online and get healthy (several services may restart during boot-up).

- Once the system is healthy, you can begin configuring the following systems in order:
    - [Configure Syncope](#configuring-syncope)
    - [Configure SonarQube](#configuring-sonarqube)
    - [More to come...](#work-in-progress)
  

### Basic Infrastructure:
#### NginX
Nginx is a reverse proxy. It'll let the outside world communicate with the inside of the docker container transparently. Nginx Requires the following volume. Here we specify a file which works just as well.
| Host                           | Container                 |
|--------------------------------|---------------------------|
| $Nginx_HOME/config/nginx.conf  | /etc/nginx/nginx.conf     |

Because the Docker containers and Docker host end up using different URLs, we have to set up a reverse proxy. But don't worry, its in docker -- all that means is that the browser and the backend apps *mostly* won't know the difference.

The proxy _should have_, for security purposes:
*TODO: need to set these up still - seems we can function in a development environment with out it.*
- An SSL certificate to deliver to clients.
- An SSL certificate to use between the services and the proxy.

But the proxy _needs_ in order to function:
- the proxy rules to map outside docker -> inside docker

These rules are defined in [Nginx's configuration volume](./volumes/config/nginx). 
I will not go into how to configure [Nginx's rules](http://nginx.org/en/docs/), the provided rules should work out of the box in a development environment. 

#### Zookeeper
Zookeeper is a service discovery tool that Syncope uses to discover itself. We don't need to configure it. 

#### Postgres 
A database. Used by both Syncope and SonarQube.

### Source Code Management and Analysis
The following tools are useful for source code development, management, analysis, and deployment.

#### [Gitlab](https://gitlab.com/gitlab-org/gitlab)
A full featured source code repository with OAuth capabilities, and even CI/CD. 

#### [SonarQube](https://github.com/SonarSource/sonarqube)
A static code analysis tool -- helps find bugs and security issues quicker. 
When built the first time, the default admin username and password are both `admin` and you will be forced to change it on the first login. I changed it to password, but you must use something stronger especially in production and test environments. 

The following volumes are required to have access to the configuration, logs, and data that sonarqube uses. 
They map from the same locations on the docker host to the service's expected location.
| Host                          | Container                 |
|-------------------------------|---------------------------|
| $Sonarqube_HOME/config        | /opt/sonarqube/conf       |
| $Sonarqube_HOME/logs          | /opt/sonarqube/log        |
| $Sonarqube_HOME/extensions    | /opt/sonarqube/extensions |
| $Sonarqube_HOME/data          | /opt/sonarqube/log        |

When sonarqube is up and running, it will be available at http://localhost:9000 -- NOTE that is not exposed via the reverse proxy, but rather the docker container itself. *Todo: I can't recall why this didn't happen beyond knowing I tried and a few issues showed up that were related to how the containers communicated with each other. Need to retry and reinvestigate.*

##### Configuring SonarQube
SonarQube can use SAML to perform AuthNZ.  Syncope needs a [Service Provider (SP) Metadata file](./volumes/syncope/config/wa/saml/sonarqube.0.xml) for SonarQube, which itself needs a certificate if you want the added security of message signing (you do in prod). This file will be loaded into Syncope, and the data in it will also be added to SonarQube. [SonarQube's documentation](https://docs.sonarqube.org/latest/instance-administration/authentication/saml/overview/) on SAML is useful. The `sonar.auth.saml.sp.privateKey.secured`, and `sonar.auth.saml.sp.certificate.secured` define the Private Key and Certificate which can be generated [quite easily](https://www.baeldung.com/openssl-self-signed-cert#creating-a-self-signed-certificate) but do note that the key needs to be converted to PKCS8 format. This step is defined below as [Syncope's SAML Prerequisite](#SAML-Prequesite) -- every SAML implementation requires this if they want to use message signing (again, you do).

We will now configure SonarQube to use Syncope as the Identity Provider (IdP).
- Login to sonarqube as Admin, and navigate to Administration > Authentication > SAML:
- Set the Application Id to `http://localhost/wa/sp/sonarqube` -- this must match what is defined in Syncope.
- Set the Provider Name to `Syncope` -- this will display on the login page.
- Set the Provider Id to ` http://localhost/wa/idp` -- this in the Issuer provided by Syncope's IdP Metadata. 
- Set the SAML Login URL to `http://localhost/wa/idp/profile/SAML2/Redirect/SSO` -- this is also defined in Syncope's IdP Metadata.
- Set Identity Provider Certificate to the value from Syncope's IdP Metadata. 
- Set SAML user login attribute to `sub`
- Set SAML user name attribute to `name`
- Set SAML user email attribute to `email`
- Set SAML group attribute to `groups`
- Ensure Sign Requests is _checked_.
- Set the Service provider private key to the PKCS8 key generated in the [Syncope + SonarQube SAML Prerequisite](#SAML-Prerequisite).
- Set the Service provider certificate to the certificate generated in the [Syncope + SonarQube SAML Prerequisite](#SAML-Prerequisite).

#### [Apache Archivia](https://github.com/apache/archiva)
An artifact repository - it stores and provides access to code artifacts such as executables, jars, etc. 

### Identity Access Management
#### [Apache Syncope](https://github.com/apache/syncope)
A full featured Identity Provider (IdP) and Identity Access Management (IAM) suite -- allowing OpenId Connect (OIDC), and Single-Sign On (SSO) for the applications deployed here. It includes several pieces: Console, Web Access (WA), Secured Remote Access (SRA), and End User. The Console is used to administer Users, Roles, and IAM generally. WA and SRA are both used to handle logins between systems each with a specific target. Enduser allows users to manage their identity as its known to the IdP. 

Some [very helpful and useful blogs about Apache Syncope](https://www.tirasa.net/en/blog/apache-syncope)
*Do note that Syncope's WA is built on [Apache CAS](https://apereo.github.io/)*

* The Console (Default credentials are `admin` and `password`)
* WA & SRA - the Access portal for SSO
* End User - An access portal where users can maintain their data in the system.
* Core - required to be running for things to work.

When running in docker, these are paths and ports the applications are accessible from. You can verify these paths in the [Nginx Config](./volumes/nginx/config/nginx.conf)
* REST API  http://localhost/syncope/rest
* Admin UI http://localhost/console * Credentials: admin / password * 
* End-user UI http://localhost/user * Credentials: syncope user based, to be set up *
* WA http://localhost/wa * Credentials: syncope user based, to be set up *
* SRA http://localhost/sra

###### Engineer's Note
Syncope has a lot of oddities, inconsistencies, and downright flaws. Sometimes errors aren't handled gracefully; some UI elements generally just "don't make sense" and the UI has variable names where display names should be used; and things will fail that need a ton of log analysis to determine what went wrong - some times that is easy, sometimes you have to turn on higher level logging and analyze flows. You can up the logging level on the Configuration > Logs page in the Console. It lets you change the level on the fly per Syncope service, and by class package path. WA is where the SAML stuff happens.
However when things are configured properly it works like a charm. 

##### Syncope Set up 
The following volumes are required to have access to the configuration, logs, and data that Syncope uses. 
*They map to the same locations on the docker container. I've found it's quite easy to copy over the default configuration from the container to the host so that Syncope can be easily configured if you accidentally lose your config. This can be achieved by temporarily targeting these directories in the container, then accessing the containers terminal via Docker, and copying all files into the new target on the container -- this will copy it out of the host and into the volume. Then return the container's target to the original value and the service will pick up the new config from the volume.*
- Syncope Core:
    | Host                          | Container           |
    |-------------------------------|---------------------|
    | $Syncope_HOME/logs/core       | /opt/syncope/log    |
    | $Syncope_HOME/config/core     | /opt/syncope/conf   |
- Syncope Console:
    | Host                          | Container           |
    |-------------------------------|---------------------|
    | $Syncope_HOME/logs/console    | /opt/syncope/log    |
    | $Syncope_HOME/config/console  | /opt/syncope/conf   |
- Syncope WA: 
    | Host                          | Container           |
    |-------------------------------|---------------------|
    | $Syncope_HOME/logs/wa         | /opt/syncope/log    |
    | $Syncope_HOME/config/wa       | /opt/syncope/conf   |
- Syncope Sra:
    | Host                          | Container           |
    |-------------------------------|---------------------|
    | $Syncope_HOME/logs/sra        | /opt/syncope/log    |
    | $Syncope_HOME/config/sra      | /opt/syncope/conf   |
- Syncope End User:
    | Host                          | Container           |
    |-------------------------------|---------------------|
    | $Syncope_HOME/logs/enduser    | /opt/syncope/log    |
    | $Syncope_HOME/config/enduser  | /opt/syncope/conf   |

Once Syncope is running and configured you can then configure SonarQube. If you want to use OpenID Connect (OIDC) instead of SAML, you will need a plugin for SonarQube as it does not support OIDC natively. Syncope, fortunately, supports several protocols.

###### Configuring Syncope
Syncope is quite large and will take some time to learn. We need to configure a few things before we're ready to use it. 
We need to define a set of rules, a set of policies, an authentication module, and a set of application configurations so that Syncope knows who SonarQube or any other application is. We will use SAML to make the connection and get the users info into the other applications.

Once docker is up and running, you may begin configuring Syncope by configuring these items: 

1. ##### Rules (Configuration > Implementations)
    - Account (Configuration > Implementations > Account_Rule)
        The Account rule defines some defaults for accounts within Syncope.
        - Click the green `+` button
        - In the [grey popup to the right](./docs/screenshots/RulesSetUp-EasyToMiss-GreyBox.png)
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
        - Click the green `+` button
        - In the [grey popup to the right](./docs/screenshots/RulesSetUp-EasyToMiss-GreyBox.png)
            - Select `Java`
            - Click the button with the Green Icon 
        - In the pop up Form:
            - Set the Key to Key to `Default Password Rule`
            - Set the Class to `org.apache.syncope.common.lib.policy.DefaultPasswordRuleConf`, again auto-filling the text area with:
            ```
            {"_class":"org.apache.syncope.common.lib.policy.DefaultPasswordRuleConf","name":"org.apache.syncope.common.lib.policy.DefaultPasswordRuleConf","maxLength":100,"minLength":3,"alphabetical":0,"uppercase":0,"lowercase":0,"digit":0,"special":0,"specialChars":[],"illegalChars":[],"repeatSame":0,"usernameAllowed":false,"wordsNotPermitted":[],"schemasNotPermitted":[]}
            ```
            - We will see this later, for now do not edit it. 
            - Click Save and now we have rules for passwords.

2. ##### Schema (Configuration > Types)
    - Schemas (Configuration > Types > Schema)
        - In the Plain section, click the green `+` button to add a new schema. 
        - In the popup menu:
            - Set the Key to `givenname`.
            - Set the Type to `String`.
            - Leave the Validator empty. 
            - Set mandatory to true.
            - Ensure Unique, Multivalue, Read-only are all unchecked. 
            - Click Next.
            - Add a new translation by clicking the green plus button, and set Locale to `en` and Display to `Given Name`
            - Click Finish
        - In the Plain section, click the green `+` button to add a new schema. 
        - In the popup menu:
            - Set the Key to `familyname`.
            - Set the Type to `String`.
            - Leave the Validator empty. 
            - Set mandatory to true.
            - Ensure Unique, Multivalue, Read-only are all unchecked. 
            - Click Next.
            - Add a new translation by clicking the green plus button, and set Locale to `en` and Display to `Family Name`.
            - Click Finish.
        - In the Derived section, click the green `+ button.
        - In the popup menu:
            - Set the key to `name`.
            - Set the Expression to `familyname + ', ' + givenname` - this is a [jexl expression](https://commons.apache.org/proper/commons-jexl/reference/) that combines the two plain schemas into a derived schema. 
            - Click Next.
            - Add a new translation by clicking the green plus button, and set Locale to `en` and Display to `Full Name`.
            - Click Finish.
    - AnyTypeClasses (Configuration > Types > AnyClasses)
        - Click the `BaseUser` row and in the grey popout to the right, click edit.
        - in the popup screen:
            - Move `familyname` and `givenname`, in the Plain Schemas section, over to the `Selected` column using the arrow buttons between them. 
            - Move `fullname` in the Derived Schemas section, over to the `Selected` column using the arrow buttons between them.
            - Click Save.    

3. ##### Policies (Configuration > Policies)
    1. Account (Configuration > Policies > Account)
        - Click the green `+` button to add a new Account Policy and:
            - Set the Name to `Default Account Policy`
            - Set the Max Authentication Attempts to something sensible like 3 or 5 - 7 if you're feeling risky.
            - Check Propagate Suspension
            - Click Save
        - Click the `Default Account Policy` item in the list.
        - Click Rules in the grey popup box on the right 
        - In the next popup, click the + button to add a new entry.
        - Then:
            - Select the Default Account Policy in the table.
            - Click Rules in the grey box to the left.
            - In the popup, click the green `+` button in the bottom left.
            - Then select `Default Account Rule` in the dropdown -- this is the rule we created above. 
            - Click next.
            - On the next screen we will see that JSON from before in a web form.
            - Set some sensible defaults - this is pretty fine-grained control, so keep it simple until its ready for production!
            - Click Finish when you're done.
    2. Password (Configuration > Policies > Password)
        - Click the + button to add a new Password Policy and:
            - Set the Name to `Default Password Policy`
            - Set the History Length to something sensible like 3 or 5 - 7 if you're feeling risky.
            - Ensure Allow Null passwords is unchecked -- unless you really want to and understand the security implications of a user with a null password. *Todo: figure out if theres a way to disable the user when the password is null.*
            - Click Save. 
        - Click the `Default Password Policy` item in the list.
        - Click Rules in the grey popup box on the right
        - In the popup screen, click the green `+` button to add a new entry. 
        - Then:
            - Select the Default Password Rule item in the dropdown - this is the rule we created above.
            - Click Next.
            - On this screen we will see that JSON from before in a web form.
            - Set some sensible defaults - this is pretty fine-grained control, so keep it simple until its ready for production!
            - Click Save.
    3. Access (Configuration > Policies > Access)
        - Click the green `+` button to add a new Access Policy and:
            - Set name to `Default Access Policy`
            - Set the Configuration to `org.apache.syncope.common.lib.policy.DefaultAccessPolicyConf`. There are others to choose from, for now this suits the needs.
            - Click Save.
        - Click the `Default Access Policy` item in the list.
        - Click Configuration in the grey popup box on the right. 
        - In the popup screen, click the green + button to add a new entry.
        - Then:
            - Set the Order to 0 so that it's first.
            - Check Enabled, SSO Enabled, Require All Attributes.
            - Ensure `Case Insensitive` is unchecked.
            - Ignore Unauthorized Redirect URL for now.
        - Click Save.
    4. Attribute Release
        - Click the green `+` button to create a new Attribute Release Policy
        - In the popup menu
            - Set the Name to `Default Attribute Release Policy`
            - Set the Order to `0`
            - Ensure Attribute Consent is _checked_ to require users consent to releasing attributes - we'll see this in action later.
            - Click Save
        - Click the new row in the table and then Configuration in the grey pop-out menu to the right.
        - In the popup screen
            - Leave Allowed Attributes, Excluded Attributes, and Include Only Attributes empty. We'll use these later.
            - Set principalIdAttr to `username`
            - Set mergingStrategy to `MULTIVALUED`
            - Ensure ignoreResolvedAttributes is unchecked. 
            - Set expiration to 5, and timeUnit to days. 
            - Leave the Selected attrRepos empty - these will come in handy later, for now Attribute Repositories are unneeded - Syncope is our repository.
    5. Authentication (Configuration > Policies > Authentication)
        - Click the green `+` button to add a new Authentication Policy and:
            - Set the Name to `Syncope Authentication Policy`
            - Click Save
        - Click the `Syncope Authentication Policy` item in the list.
        - Click Configuration in the grey popup box on the right.
        - In the popup Screen, Syncope should be in the Available list. Click the arrow point to the Selected list to tell Syncope to use it for this policy. Fret not if it is not available as that just means you haven't created an [authentication module](#authentication-module-wa--authentication-modules) yet - which should be the next step. When you do, come back here and set the Module in the policy. 
        
4. ##### Authentication Module (WA > Authentication Modules)
    This defines our "method of authentication." More specifically it defines a way someone can authenticate to Syncope - in this case, we will use Syncope as the method but there are others available which require different settings.
    - Click the green `+` button to add a new Authentication Module.
    - Set the Key to `Syncope Authentication Module`
    - Ignore Description (unless you really want to add it).
    - Set the State to Active.
    - Set the order to 0.
    - Set the Type to `org.apache.syncope.common.lib.auth.SyncopeAuthModuleConf`.
    - Click Next.
    - Set the Domain (which is also known as a Realm) to Master.
    - Click Next. 
    - Add the following mapping of attributes:
        | Internal attribute           | External attribute | Did we make? |
        |------------------------------|--------------------|--------------|
        | syncopeUserAttr_fullname     | name               | ✅ (Schema)  |
        | syncopeUserAttr_email        | email              | ❌           |
        | syncopeUserKey               | sub                | ❌           |
        | syncopeUserDynMemberships    | groups             | ❌           |
        | syncopeUserMemberships       | groups             | ❌           |
        | syncopeUserStatus            | user-status        | ❌           |
        | username                     | username           | ❌           |
    - Click "Finish"

5. ##### Syncope's IdP (WA > SAML 2.0 > Identity Provider)
    Syncope's Identity Provider needs to be configured properly for the environment its running in. This means updating the IdP metadata to correct URL from the default. If you see errors in WA's logs (you may need to up the log level) about the issuer being invalid, this is what you need to do. You will see this error if you don't.
    - Click the Syncope row with `http://syncope-wa:8080/wa/idp/metadata` as the url.
    - In the grey pop-out menu to the right click Edit.
    - In the popup screen you will see an XML file. There are several edits we need to make to this so SAML will work.
        - Update the entityId to equal `http://localhost/wa/idp`
        - For a production environment, you'll want to also change the instances of `SingleSignOnService` and `SingleLogoutService` elements that contain `http://syncope-wa:8080` which is the hostname and port only available inside the container. I was able to skip this for development.

6. ##### Group (Realms > Group)
    - Click the green `+` button to add a new Group.
    - In the popup screen:
        - set Destination realm to `/` which is also know as the Master or root realm.
        - Set the name to `sonar-administrators`. This group will, when Sonarqube sees it, will make Sonarqube give our users the correct permissions.
        - Click Next. 
        - The next screen is used to set an owner of the group which, at this point, we don't need. So, click Next.
        - The next screen is used to add users into the group when its created based on a range of conditions. For now, as there shouldn't be any users in the system yet, click Next.
        - On this next screen click Finish - we have created a group. Now lets put a user into it.

7. ##### User (Realms > User)
    - Click the green `+` button to add a new User.
    - In the popup screen: 
        - Set Destination realm to `/` which is also know as the Master or root realm.
        - Set Username to your favorite username.
        - Click `Password Management` and set the password for this user.
        - Click Next.
        - On this next screen, move `BaseGroup` over to the `Selected` column using the arrow buttons between them. 
        - Click Next.
        - Move the `sonar-administrators` group to the `Selected` column using the arrow buttons between them.  Ignore Dynamic groups and Dynamic realms.
        - Click Next.
        - Set the email your favorite email address.
        - Set the other fields here to your liking then click Next.
        - Click next. This page is just showing us the derived attributes 
        - We do not yet need to work with roles, but we will in the future. So for now click Finish. Wer have a user to sign in with at this point and that can be tested through the Enduser and WA applications by navigating to them and logging in.  

8. ##### SonarQube SAML (WA > Client Applications > SAML2SP)
    ###### SAML Prerequisite
    We must generate a private key, a certificate, and a SAML Service Provider's metadata file. The certificate and key will be loaded into SonarQube when we configure it's end of the SAML protocol. Syncope will also need the certificate. We'll also need to create a PKCS8 formatted version of the private key. These files must be securely transmitted and stored - they are _critical_ to the security of the SAML authentication process.
    Once the cert, and the 2 versions of the key are in place, you can [create a SAML metadata for the service provider](https://www.samltool.com/sp_metadata.php) *Beware: private keys added there are sent to their server. Trust at your own risk and do not use it for production keys.  
    
    ###### Protect these files like you protect your identity because that's literally what these files are - the identity of SonarQube as far as Syncope is concerned. If these are leaked then the authentication process is no longer secure. In production store them in a secrets manager. 
    ```
    $ openssl req -x509 -newkey rsa:4096 -keyout sonarqube.key -out sonarqube.crt -days 365 # generate a new key and cert that'll last for a year.
    $ openssl pkcs8 -topk8 -in sonarqube.key -out sonarqube.pkcs8.key -nocrypt # create PKCS8 version of the private key
    ```
    [Example Metadata file](./volumes/syncope/config/wa/saml/sonarqube.0.xml) This file is the file used by Syncope to validate the data SonarQube sends it when doing a SAML exchange. All Service Provider metadata files will look similar with different values for various field such as `EntityId` and `validUntil`. 
    ```
        Double check your validUntil field -- it must be a future date.
    ```
    - Have the Prerequisites above in place. 
    - Click the Green Plus Icon and in the Popup menu:
        - Set the Realm to `\`.
        - Set the Name to `sonarqube`.
        - Set the ID to 0.
        - Set the Description to nothing or whatever you want.
        - Set the Access Policy to the one we defined earlier `Default Access Policy`.
        - Set the Attribute Release Policy to the one we defined earlier `Default Attribute Release Policy`.
        - Set the Authentication Policy to the one we defined earlier `Default Authentication Policy`.
        - Leave the Theme blank.
        - Set the EntityId to `http://localhost/wa/sp/sonarqube` or whatever you've defined in SonarQube's metadata.
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

9. ##### SonarQube OIDC (WA > Client Applications > OIDCRP) *Optional -- requires a plugin to be installed into SonarQube. The OIDC configuration for Sonarqube is similar to SAML with some differences. We will not go into configuring Sonarqube's end of OIDC, but do go over Syncope's as the process is the same for other applications.
    - Set the realm to `/`
    - Set the name to Sonarqube (or some other app name)
    - Set the ID to 1
    - Ignore Description (unless you really want to add it)
    - Set the Access to policy to the one created earlier.
    - Ignore the Attribute Release Policies
    - Set the Authentication Policy to the one created earlier.
    - Set the Client ID and Secret -- REMEMBER THESE VALUES, you'll need them later to set up the OIDC IDP in the other application -- in this case its SonarQube and thats the  app that'll get these values, but you could very well have another application defined here and it must use its own values for these fields. This is how Syncope "knows" the application and allows AuthNZ to occur through it.
    - Check `Sign IdToken`, `JWT Access Token`, and `Bypass Approval Prompt`
    - Set the Subject Type to `PUBLIC`
    - Set the `redirect_uri` to `http://localhost:9000/oauth2/callback/oidc` if you're setting up sonarqube, or to the correct url if you're setting up a different application.
    - Add all Supported Grant Types and Supported Response Types
    - Ignore Logout Uri

##### Production Readiness: 
    To harden your Syncope system and get it ready for production:
* Note the changes mentioned else where for production, and change those.
* Change the default admin password and force its rotation every month; or disable the user.
* Remove Unused, weak, or low-bit algorithms for the OIDC, OAUTH2, SAML, and other auth protocols. 
* Modify the logging configuration to not emit tokens.
* Turn off the self-registration in the Enduser app; or otherwise control when, what, and how users can self-register. 
* Anywhere `localhost` shows up will need changing to be the production domain. The Identity Provider metadata file is one starting place for that - but many configurations will need to be changed too.  


#### Cloud Application Frameworks
##### [Apache ServiceComb](https://servicecomb.apache.org/)
A microservice tool - it lets you set up and run microservices.

##### [Apache OpenWhisk](https://github.com/apache/openwhisk)
A serverless function platform. 

##### [Apache Karaf](https://github.com/apache/karaf)
An full featured application runtime playing a similar role as ServiceComb

##### [Apache Skywalking](https://github.com/apache/skywalking)
An Application Performance Monitoring tool. Allows devs, leaders, and the whole team see how each of your Applications are performing. 

##### [Apache Kafka](https://github.com/apache/kafka)
A Pub-Sub message broker. Allows ingestion of notifications to and from systems. 





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
Think WebEx or Zoom -- but yours and free.





#### Honorable Mentions
##### [Apache Cloudstack](https://github.com/apache/cloudstack)
Cloudstack is a pretty interesting, well known, and highly used cloud solution. However, running it in Docker seems counter-intuitive as it's better when on bare metal.
##### [Apache Allura](https://github.com/apache/allura)
Allura is a neat software forge but many of the features it has is also offered by gitlab - and many that aren't. It may still make it in...



##### Work In Progress
This is a work in progress that is not yet, and may never be especially without proper configuration, ready for production deployments. Many of the applications listed here may or may not make it into the final Box. But there is a vision and a plan; and this is still a work in progress.

The plan is to include Gitlab, Archivia, and one of ServiceComb or OpenWhisk; followed by the business applications; and then the rest of Cloud Application Frameworks for future applications. 