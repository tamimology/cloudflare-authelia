# Cloudflare Tunnel Login Using Self-hosted Authelia

# Table of Contents

First, we start with Cloudflare platform

1. <a href="#generating-service-tokens">Generating Service Tokens</a>
2. <a href="#setting-up-authentication-methods">Setting Up Authentication Methods</a>
3. <a href="#adding-applications-rules-and-policies">Adding Applications Rules and Policies</a>
4. <a href="#modifying-proxies-and-fowarded-headers">Modifying Proxies and Fowarded-headers</a>


Second, we move to Authelia self-hosted

1. <a href="#generating-secrets-and-openid-issuer-private-key">Generating Secrets and OpenID Issuer Private Key</a>
2. <a href="#generating-random-alphanumeric-string">Generating Random Alphanumeric String</a>
3. <a href="#editing-authelia-configuration.yml-file">Editing Authelia configuration.yml file</a>

Finally, we test our setup.




## Generating Service Tokens


To begin with, open the [Cloudflare Zero Trust](https://dash.teams.cloudflare.com/) page. Then from the left hand list, expand the **Access** drop down, and select the **Service Auth** from that drop down list.

From the right hand side, click on **Create Service Token**

![service token](/screenshots/service_token.png)

Give the token a name, i.e. *Authelia*, and set the duiration to *Non-expiring* and click **Generate token**

Now, 2 CF-Access-Client will be generated. **COPY THEM AS THIS IS THE ONLY TIME YOU WILL BE ABLE TO SEE THEM** and have them saves in a safe place for later usage

![token ids](/screenshots/token_ids.png)


## Setting Up Authentication Methods

On the left side, click on **Settings**, then choose **Authentication**

![authentication](/screenshots/authentication.png)


In the next window, under the *Login methods*, click **Add new**, and then Choose the **OpenID Connect** from the available options

![new authentication](/screenshots/new_authentication.png)


*Now, this has to be filled in carefully and remeber each entry for later use.*

> 1- Fill in the name, i.e. **Authelia**

Considering the URL for Authelia as *auth.mydomain.com*, then in each field that follows enter:

> 2-**CF-Access-Client-Id** generated before

> 3-**CF-Access-Client-Secret** generate before

> 4-https://auth.mydomain.com/api/oidc/authorization

> 5-https://auth.mydomain.com/api/oidc/token

> 6-https://auth.mydomain.com/jwks.json

> 7-Enable the *Proof Key for Code Exchange (PKCE)

> 8-add new OIDC Claims, *preferred_username* and *mail*

Click on Save

![openid connect](/screenshots/openid_connect.png)



## Adding Applications Rules and Policies

On the left side, click on **Access**, then choose **Applications**, then click on **Add an application** and choose the **Self-hosted** option.

![add application](/screenshots/add_application.png)

In the **Configue app** screen, give the applicaiton a name, i.e. **Portainer-CE**. Choose subdomain, i.e. **portainer**, and choose your domain, which we agreed on to be **mydomain.com**. You can change the application logo if needed, or jsut keep the *Default* one. Click on **Next** on the top right.

In the **Add policies** screen, give this policy a name, i.e. "*portainer_auth_policy*". In the **Action**, select the "*Allow*", and duration make it "*1 month*".
Then in **Configure rules**, set the **Selector** to "*Login Methods*", and the **Value**, choose the "*OpenID Connect Authelia*" option and click on **Next** on the top right.

In the **Setup** screen, leave everything as is, and click on **Add application** on the top right


![application access](/screenshots/application_access.png)







## Modifying Proxies and Fowarded-headers

Now, go back to the [Cloudfare dashboard](https://dash.cloudflare.com/), click on the **domain** from the *Home* screen. On the left list, scroll down till you find the **Rules**, expand it and choose **Transform Rules** from the drop list. 
Now from there, click on **Create transform rule** and choose the **Modify Request Header**

![transform rules](/screenshots/transform_rules.png)

Give the rule a name, i.e. **Authelia**. In the "*Field*", choose the "*X-Forwarded-For*" option, "*Operator*" to be "*does not equal*", and leave the "*Value*" as *blank*
Add another Rule by clicking on the *And*, and In the "*Field*", choose the "*IP Source Address*" option, "*Operator*" to be "*is not in*", and in the "*Value*", enter the locally hosted ***Cloudflare Tunnel Container IP address***.
Below, in the **Then...** part, choose "*Remove*", and in the "*Header name*", enter "*X-Forwarded-For*" and click on **Save**

![proxy header](/screenshots/proxy_header.png)









Now to make sure that, so far, everything is set up correctly, click on **Test** beside the newly created *Login Method*. If the below is seen, then it is good to proceed with the next steps, otherwise, check what is done wrong

![openid test](/screenshots/openid_test.png)
