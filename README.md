# Cloudflare Tunnel Login Using Self-hosted Authelia

#### **NOTE:** *WHEN WRITING THIS GUIDE, IT WAS BASED ON [AUTHELIA v4.36.9](https://github.com/authelia/authelia) AND [CLOUDFLARE TUNNEL v2022.10.2](https://github.com/cloudflare/cloudflared)*


# Table of Contents

***First, we start with Cloudflare platform***

1. <a href="#generating-service-tokens">Generating Service Tokens</a>
2. <a href="#setting-up-authentication-methods">Setting Up Authentication Methods</a>
3. <a href="#adding-applications-rules-and-policies">Adding Applications Rules and Policies</a>
4. <a href="#modifying-proxies-and-fowarded-headers">Modifying Proxies and Fowarded-headers</a>


***Second, we move to Authelia self-hosted***

5. <a href="#generating-secrets-and-openid-issuer-private-key">Generating Secrets and OpenID Issuer Private Key</a>
6. <a href="#generating-random-alphanumeric-string">Generating Random Alphanumeric String</a>
7. <a href="#editing-authelia-configuration-file">Editing Authelia Configuration File</a>

***Finally, we test our setup***

8. <a href="#testing-the-integration-setup">Testing The Integration Setup</a>



## Generating Service Tokens

#### This step will be used in a later step as the link where both Authelia and Cloudflare are integrated together.

To begin with, open the [Cloudflare Zero Trust](https://dash.teams.cloudflare.com/) page. Then from the left hand list, expand the **Access** drop down, and select the **Service Auth** from that drop down list.

From the right hand side, click on **Create Service Token**

![service token](/screenshots/cloudflare/service_token.png)

Give the token a name, i.e. "***Authelia***", and set the duration to ***Non-expiring*** and click **Generate token**

Now, two *CF-Access-Client* will be generated. **COPY THEM AS THIS IS THE ONLY TIME YOU WILL BE ABLE TO SEE THEM** and have them saved in a safe place for later usage

![token ids](/screenshots/cloudflare/token_ids.png)


## Setting Up Authentication Methods

#### This step is where we add the Authelia as a 2FA service into Cloudflare platform.

On the same page you are now, and on the left side, click on **Settings**, then choose **Authentication**

![authentication](/screenshots/cloudflare/authentication.png)


In the next window, under the ***Login methods***, click **Add new**, and then Choose the **OpenID Connect** from the available options

![new authentication](/screenshots/cloudflare/new_authentication.png)


*Now, this has to be filled in carefully and remember each entry for later use.*

1. Fill in the name, i.e. "***Authelia***"

Considering the URL for Authelia as "***auth.mydomain.com***", then in each field that follows enter:

2. ***CF-Access-Client-Id*** generated in the previous step
4. ***CF-Access-Client-Secret*** generate in the previous step
5. https://***auth.mydomain.com***/api/oidc/authorization
6. https://***auth.mydomain.com***/api/oidc/token
7. https://***auth.mydomain.com***/jwks.json
8. Enable the ***Proof Key for Code Exchange (PKCE)***
9. Add new **OIDC Claims**, ***preferred_username*** and ***mail***

Click on **Save**

![openid connect](/screenshots/cloudflare/openid_connect.png)


## Adding Applications Rules and Policies

#### This is where you define for each subdomain, if Authelia is needed to be an intermediate gate between the Cloudflare Tunnel and the final URL requested

On the left side, click on **Access**, then choose **Applications**, then click on **Add an application** and choose the **Self-hosted** option.

![add application](/screenshots/cloudflare/add_application.png)

In the **Configure app** screen, give the applicaiton a name, i.e. "***Portainer-CE***". Choose subdomain, i.e. "***portainer***", and choose your domain, which we agreed on to be ***mydomain.com***. You can change the application logo if needed, or just keep the *Default* one. Click on **Next** on the top right.

In the **Add policies** screen, give this policy a name, i.e. "***portainer_auth_policy***". In the **Action**, select the "***Allow***", and duration make it "***1 month***".
Then in **Configure rules**, set the **Selector** to "***Login Methods***", and the **Value**, choose the "***OpenID Connect Authelia***" option and click on **Next** on the top right.

In the **Setup** screen, leave everything as is, and click on **Add application** on the top right


![application access](/screenshots/cloudflare/application_access.png)


## Modifying Proxies and Fowarded-headers
#### Cloudflare, bu defualt, adds the *X-Forwarded-For* header, or it appends another IP, depending if this header is originally there during the request or not. Therefore, this has to be modified to remove the IP address so that the client sending this request will not be able to forge that remote IP address with any of the most accepted remote IP headers.

Now, go back to the [Cloudfare dashboard](https://dash.cloudflare.com/), click on the **domain** from the *Home* screen. On the left list, scroll down till you find the **Rules**, expand it and choose **Transform Rules** from the drop list. 
Now from there, click on **Create transform rule** and choose the **Modify Request Header**

![transform rules](/screenshots/cloudflare/transform_rules.png)

Give the rule a name, i.e. "***Authelia***". In the "***Field***", choose the "*X-Forwarded-For*" option, "***Operator***" to be "*does not equal*", and leave the "***Value***" as *blank*
Add another Rule by clicking on the "**And**", and In the "***Field***", choose the "*IP Source Address*" option, "***Operator***" to be "*is not in*", and in the "***Value***", enter the locally hosted ***Cloudflare Tunnel Container IP address***.
Below, in the **Then...** part, choose "***Remove***", and in the "***Header name***", enter "*X-Forwarded-For*" and click on **Save**

![proxy header](/screenshots/cloudflare/proxy_header.png)


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

As a quick summary, just delete the part for the "*access_control:*", and keep the rest untouched. Also make sure that you change all "*URLs*" from the previous one to the Cloudflare's ones, i.e. from "*DuckDNS*"

Considering the team's name in Clkoudflare is named as "***myteam***", then, we need to add the following at the end of the file:

```yaml
identity_providers:
  oidc:
    hmac_secret: **PUT THE GENERATED RSA STRING FROM THE PREVIOUS STEP**
    issuer_private_key: |
      -----BEGIN RSA PRIVATE KEY-----
      **HERE GOES WHATEVER IS IN THE _private.pem_ FILE**
      -----END RSA PRIVATE KEY-----
    access_token_lifespan: 1h
    authorize_code_lifespan: 1m
    id_token_lifespan: 1h
    refresh_token_lifespan: 90m
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
      - id: **PUT THE _CF-Access-Client-Id_ TOKEN PREVIOUSLY GENERATED **
        description: Cloudflare ZeroTrust
        secret: 8**PUT THE _CF-Access-Client-Secret_ TOKEN PREVIOUSLY GENERATED **
        public: false
        authorization_policy: two_factor
        pre_configured_consent_duration: '365d'
        redirect_uris:
          - https://**myteam**.cloudflareaccess.com/cdn-cgi/access/callback
        scopes:
          - openid
          - profile
          - email
        userinfo_signing_algorithm: none
```

Now, restart Authelia container and make sure that no errors are in the logs. If this is successfull, it is time now to test the integration from **Cloudflare**.


## Testing The Integration Setup

#### This is the last step to make sure that nothing wrong is made during the setup, and *Authelia* is successfully integrated into *Cloudflare* as a 2FA platform

Now to make sure that, so far, everything is set up correctly, open the [Cloudflare Zero Trust](https://dash.teams.cloudflare.com/) page. Click on **Settings**, then **Authentication**. Under the ***Login methods*** you will see the previously added "*OpenID Connect Authelia*" method. Click on 
**Test** beside it.

![authentication test](/screenshots/cloudflare/authentication_test.png)


If the below is seen, then *Authelia* is now a gateway for your *Cloudflare*'s selected domains for 2FA authentication. Otherwise, re-check what have you missed from this guide, as it is 100% guaranteed if followed as is, the integration will be successful.

![openid test](/screenshots/cloudflare/openid_test.png)
