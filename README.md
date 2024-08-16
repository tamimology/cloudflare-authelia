# Cloudflare Tunnel Login Using Self-hosted Authelia

#### **NOTE:** *WHEN WRITING THIS GUIDE, IT WAS BASED ON [AUTHELIA v4.38.10](https://github.com/authelia/authelia) AND [CLOUDFLARE TUNNEL v2024.8.2](https://github.com/cloudflare/cloudflared)*


# Table of Contents

***First, we start with Cloudflare platform***

1. <a href="#generating-service-tokens">Generating Service Tokens</a>
2. <a href="#generating-hashed-secrets">Generating Hashed Secrets</a>
3. <a href="#setting-up-authentication-methods">Setting Up Authentication Methods</a>
4. <a href="#adding-applications-rules-and-policies">Adding Applications Rules and Policies</a>
5. <a href="#adding-firewall-rules">Adding Firewall Rules</a>


***Second, we move to Authelia self-hosted***

6. <a href="#generating-secrets-and-openid-issuer-private-key">Generating Secrets and OpenID Issuer Private Key</a>
7. <a href="#generating-random-alphanumeric-string">Generating Random Alphanumeric String</a>
8. <a href="#editing-authelia-configuration-file">Editing Authelia Configuration File</a>

***Finally, we test our setup***

9. <a href="#testing-the-integration-setup">Testing The Integration Setup</a>



## Generating Service Tokens

#### This step will be used in a later step as the link where both Authelia and Cloudflare are integrated together.

To begin with, open the [Cloudflare Zero Trust](https://dash.teams.cloudflare.com/) page. Then from the left hand list, expand the **Access** drop down, and select the **Service Auth** from that drop down list.

From the right hand side, click on **Create Service Token**

![service token](/screenshots/cloudflare/service_token.png)

Give the token a name, i.e. "***Authelia***", and set the duration to ***Non-expiring*** and click **Generate token**



## Generating Hashed Secrets

#### This step will be used in a later step as the link where both Authelia and Cloudflare are integrated together.

In the terminal, execute the command `docker run authelia/authelia:latest authelia crypto hash generate pbkdf2 --variant sha512 --random --random.length 72 --random.charset rfc3986` and take note of the both the *Random Password* and *Digest* outputs.



## Setting Up Authentication Methods

#### This step is where we add the Authelia as a 2FA service into Cloudflare platform.

On the same page you are now, and on the left side, click on **Settings**, then choose **Authentication**

![authentication](/screenshots/cloudflare/authentication.png)


In the next window, under the ***Login methods***, click **Add new**, and then Choose the **OpenID Connect** from the available options

![new authentication](/screenshots/cloudflare/new_authentication.png)


*Now, this has to be filled in carefully and remember each entry for later use.*

1. Fill in the name, i.e. "***Authelia***"

Considering the URL for Authelia as "***auth.mydomain.com***", then in each field that follows enter:

2. ***cloudflare***
4. The ***Random Password*** generate in the previous step
5. https://***auth.mydomain.com***/api/oidc/authorization
6. https://***auth.mydomain.com***/api/oidc/token
7. https://***auth.mydomain.com***/jwks.json
8. Enable the ***Proof Key for Code Exchange (PKCE)***
9. Add new **OIDC Claims**, ***preferred_username*** and ***mail***

Click on **Save**

![openid connect](/screenshots/cloudflare/openid_connect.png)


Before leaving the **Cloudflare** platform and moving to the **Authelia** part, take note of the **Cloudflare** ***team*** name, and have it saved somewhere safe for later use. To do that, click on **Settings** on the left side list, then choose **Custom Pages**. On the top right side, you will see the team name in the format *myteam.cloudflareaccess.com*. So, in this case, it will be only ***myteam**



![cf team](/screenshots/cloudflare/cf-team.png)


## Adding Applications Rules and Policies

#### This is where you define for each subdomain, if Authelia is needed to be an intermediate gate between the Cloudflare Tunnel and the final URL requested

On the left side, click on **Access**, then choose **Applications**, then click on **Add an application** and choose the **Self-hosted** option.

![add application](/screenshots/cloudflare/add_application.png)

In the **Configure app** screen, give the application a name, i.e. "***Portainer-CE***". Choose subdomain, i.e. "***portainer***", and choose your domain, which we agreed on to be ***mydomain.com***. You can change the application logo if needed, or just keep the *Default* one. Click on **Next** on the top right.

In the **Add policies** screen, give this policy a name, i.e. "***portainer_auth_policy***". In the **Action**, select the "***Allow***", and duration make it "***1 month***".
Then in **Configure rules**, set the **Selector** to "***Login Methods***", and the **Value**, choose the "***OpenID Connect Authelia***" option and click on **Next** on the top right.

In the **Setup** screen, leave everything as is, and click on **Add application** on the top right


![application access](/screenshots/cloudflare/application_access.png)

## Adding Firewall Rules

#### What we need now is to allow specific countries to access the domain, and block all other countries to avoid external attacks.

In the [Cloudfare dashboard](https://dash.cloudflare.com/), click on the **Security** from the *Home* screen, then choose **WAF**. On the right side, click on ***Create rule***.


![waf](/screenshots/cloudflare/waf.png)

Follow the steps as below:
1. Enter the desired Rule Name
2. Choose *"Continent"* from the **Field** list, **Operator** to be *"equals"*, and then choose the first option in the **Value** list. Do this step for all items in the **"Value"** list by choosing the *"Or"* next to each rule line
3. Choose the *"Block"* from **action**

![waf_rule](/screenshots/cloudflare/waf-rule.png)

By doing this, you have blocked all countries to accessing your domain.


Now, you need to unblock your country (or more than one if needed).

After you have added all continents as block, check your desired country is in which continent, i.e. Cyprus, which is in Europe. You go to the Europe option you have added, and then click on **And**, then choose *"Country"* from the **Field** list, **Operator** to be *"does not equals"*, and then choose *"Cyprus"* from the **Value** list.

Click on **Deploy firewall rule** at the bottom of the page

Your final result shoul look like below

![final_waf](/screenshots/cloudflare/final-waf.png)


###### In case you have Home Assistant installed, and have the Google Home Devices integrated with it, you need to allow Google to bypass the firewall rule. In order to do that, with the **North America** rule, add an **And** rule, then choose *"AS Num"* from the **Field** list, **Operator** as *"does not equal"* and set the **Value** to *"15169"*


###### In case you have MariaDB installed, you need to allow it to bypass the firewall rule. In order to do that, with the **North America** rule, add an **And** rule, then choose *"AS Num"* from the **Field** list, **Operator** as *"does not equal"* and set the **Value** to *"396982"*


## Generating Secrets and OpenID Issuer Private Key

#### What we need now is to generate an encrypted pair of RSA file to be used to encrypt Authelia public requests.

To do so, SSH into the host machine having the Authelia container installed on, and considering that:
*UID="1000"*, *GID="100"* , and *PEM file output directory="/rsa"*

Enter and execute the following command: `docker run -u 1000:100 -v /rsa:/keys authelia/authelia:latest authelia crypto pair rsa generate --bits 4096 --directory /keys`

2 `.pem` files will be generated in the "**/rsa**" folder defined in the command executed

![pem files](/screenshots/authelia/pem_files.png)

We are only in need of the **private.pem** file, keep it somehwere safe for later use.


## Generating Random Alphanumeric String

#### This step is to generate an alphanumeric string to be used as a secret hash for Authelia when accessing it from Cloudflare

In the same SSH terminal, execute the following command `tr -cd '[:alnum:]' < /dev/urandom | fold -w "64" | head -n 1 | tr -d '\n' ; echo`

An output string will be shown on the terminal window, also copy that and keep in a safe place for later use.

![rsa](/screenshots/authelia/rsa.png)


## Editing Authelia Configuration File

#### Because Cloudflare has different settings to be configured, and are different compared to the one previously used with any reverse proxy, a new configuration.yml file should be created with the relevant parameters in it

In case of an existing "*configuration.yml*" file is available in the Authelia folder, make a backup copy of that, or simply download the sample one from [here](/configuration.yml) and make sure to change all CAPITALISED parameters to match your setup

As a quick summary, just delete the "*rules:*" part under the "*access_control:*", and keep the rest untouched. Also make sure that you change all "*URLs*" from the previous one to the Cloudflare's ones, i.e. from "*DuckDNS*"

Considering the team's name in Clkoudflare is named as "***myteam***", then, we need to add the following at the end of the file:

```yaml
identity_providers:
  oidc:
    hmac_secret: **PUT THE GENERATED RSA STRING FROM THE PREVIOUS STEP**
    jwks:
      - key_id: 'example'
        algorithm: 'RS256'
        use: 'sig'
        key: | # docker run -u 1036:100 -v /volume1/docker:/keys authelia/authelia:latest authelia crypto pair rsa generate --bits 4096 --directory /keys
          -----BEGIN RSA PRIVATE KEY-----
          **HERE GOES WHATEVER IS IN THE _private.pem_ FILE**
          -----END RSA PRIVATE KEY-----
    lifespans.access_token: 1h
    lifespans.authorize_code: 1m
    lifespans.id_token: 1h
    lifespans.refresh_token: 90m
    enable_client_debug_messages: false
    enforce_pkce: public_clients_only
    cors:
      endpoints:
        - authorization
        - token
        - revocation
        - introspection
      allowed_origins:
        - "*"
      allowed_origins_from_client_redirect_uris: false

    clients:
      - client_id: cloudflare
        client_name: Cloudflare ZeroTrust
        client_secret: **PUT THE _Digest_ PASSPHRASE PREVIOUSLY GENERATED **
        public: false
        authorization_policy: two_factor
        pre_configured_consent_duration: '365d'
        redirect_uris:
          - https://**myteam**.cloudflareaccess.com/cdn-cgi/access/callback
        scopes:
          - openid
          - profile
          - email
        userinfo_signed_response_alg: none
```

Now, restart Authelia container and make sure that no errors are in the logs. If this is successful, it is time now to test the integration from **Cloudflare**.


## Testing The Integration Setup

#### This is the last step to make sure that nothing wrong is made during the setup, and *Authelia* is successfully integrated into *Cloudflare* as a 2FA platform

Now to make sure that, so far, everything is set up correctly, open the [Cloudflare Zero Trust](https://dash.teams.cloudflare.com/) page. Click on **Settings**, then **Authentication**. Under the ***Login methods*** you will see the previously added "*OpenID Connect Authelia*" method. Click on 
**Test** beside it.

![authentication test](/screenshots/cloudflare/authentication_test.png)


If the below is seen, then *Authelia* is now a gateway for your *Cloudflare*'s selected domains for 2FA authentication. Otherwise, re-check what have you missed from this guide, as it is 100% guaranteed if followed as is, the integration will be successful.

![openid test](/screenshots/cloudflare/openid_test.png)





## License
This document guide is licensed under the CC0 1.0 Universal license. The terms of the license are detailed in [LICENSE](/LICENSE)
