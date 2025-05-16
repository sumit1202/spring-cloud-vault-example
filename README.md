## Using VaultConfigurer to override authentication while still using Vault for config.

Is authentication: custom a valid Spring Cloud Vault setting?
Yes — authentication: custom is valid in Spring Cloud Vault, but only in a specific context:

It means:
“Do not use any built-in authentication method, I’ll provide a custom one via VaultConfigurer.”

When is it valid?
When you register a @Bean of type VaultConfigurer (or implement AbstractVaultConfiguration), Spring will respect your custom ClientAuthentication.

authentication: custom tells Spring not to auto-configure its built-in ones like token, aws_iam, approle, etc.

🚫 If you don’t provide VaultConfigurer and set authentication: custom
Then you’ll get an error like:

No ClientAuthentication provided and authentication method 'custom' is not supported.

1: In bootstrap.yml, spring.cloud.vault.authentication=custom -> tells Spring Cloud Vault not to use built-in auth like token, kubernetes, or approle.

Spring Cloud Vault invokes the custom VaultConfigurer bean on bootstrap.

2: Provide a VaultConfigurer implementation where you override the clientAuthentication() method.

3: Custom UamiVaultClientAuthentication runs
This class does two things:

(a) Calls Azure IMDS to get a UAMI token:
This returns a JWT access token issued by Azure AD for the user-assigned managed identity (UAMI) tied to your App Service.
(b) Calls Vault's Azure login endpoint:
Vault validates this token with Azure AD. If valid, Vault returns a Vault client token like:

4: Vault Token is returned to Spring Cloud Vault
Spring Cloud Vault now uses this token to fetch secrets from the configured path (e.g., secret/data/application).
You can access these secrets like:

✅ Step 5: Spring finishes bootstrapping with Vault-sourced configuration
Secrets from Vault are injected into the environment during the bootstrap phase. You can use them via:
@Value
@ConfigurationProperties
Spring Config server (if applicable)

🔐 Why this works in Azure App Service with UAMI
Azure App Service exposes a Metadata endpoint (IMDS) (169.254.169.254) which supports UAMI-based token issuance.

Vault has an Azure auth method that supports JWT tokens from UAMI.
Spring Cloud Vault allows full control of auth via VaultConfigurer.

