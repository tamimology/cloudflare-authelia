# Cloudflare Tunnel Login Using Self-hosted Authelia

# Table of Contents

First, we start with Cloudflare platform

1. <a href="#tokens">Generating Authelia Service Tokens</a>
2. <a href="#cf-auth">Setting Up Authentication Methods</a>
3. <a href="#proxy-headers">Modifying proxies/fowarded-headers</a>
4. <a href="#rules-policies">Adding Applicaitons Rules/Policies</a>


Second, we move to Authelia self-hosted

1. <a href="#secrets">Generating Secrets/OpenID Issuer Private Key</a>
2. <a href="#rsa-gen">Generating Random Alphanumeric String</a>
3. <a href="#authelia-config">Editing Authelia configuration.yml file</a>

Finally, we test our setup.
