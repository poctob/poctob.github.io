---
layout: project
title: Dotnet core configuration and options pattern
description: Simple recipie of options pattern common use
summary: 
category: Dotnet core recipes
---
- [Simple IConifguration Setup](#simple-iconfiguration-based-setup-if-you-need-to-access-one-or-two-string-based-values)
- [Configuration to POCO and Options patern](#standard-configuration-to-poco-and-options-pattern)
- [Key Vault based configuration](#using-azure-key-vault-to-manage-and-access-sensitive-configuration-values)
- [Configuration file in Azure Blob Storage](#storing-and-accessing-encrypted-configuration-file-in-azure-blob-storage)

---
## Simple IConfiguration based setup, if you need to access one or two string based values
1. Add configuration data into `appsettings.json`
```json
{
    "FileConfig": {
      "FileName": "myfile.txt",
      "Owner": "John Doe"
    }
}
```
2. Access configuration values using IConfiguration
```cs
    public class Startup
    {
        public IConfiguration Configuration { get; }

        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public void ConfigureServices(IServiceCollection services)
        {
            //Access config value
            string fileName = Configuration["FileConfig:FileName"];

            //Another way to access configuration value
            string fileOwnere = Configuration.GetSection("FileConfig")["Owner"];
        }
    }
```

---
## Standard configuration to POCO and options pattern
1. Create POCO representation of your configuration:
```cs
namespace MyProject.Config
{
    public class FileConfig
    {
        public int MaxFileSize {get; set;}
    }
}
```
2. Add configuration data into `appsettings.json`
```json
{
    "FileConfig": {
      "MaxFileSize": 10000000
    }
}
```
3. Bind configuration to POCO model in `Startup.cs` 
```cs
    public void ConfigureServices(IServiceCollection services)
    {
        services.Configure<FileConfig>(options => Configuration.GetSection("FileConfig").Bind(options));
    }
```
4. Define and inject conifguration in your service or controller
```cs
    using Microsoft.Extensions.Options;
    namespace MyProject.Config
    {
        public class FileService
        {
            private readonly FileConfig fileConfig;
            
            public FileService(IOptions<FileConfig> fileConfig)
            {
                this.fileConfig = fileConfig.Value;
            }
        }
    }
```

5. Use configuration as any POCO object
```cs
       
        public bool CheckFileSize(int size)
        {
            return size > fileConfig.MaxFileSize;
        }
```

---
## Using Azure Key Vault to manage and access sensitive configuration values

### Ingredients

- Packages
  - Microsoft.Azure.KeyVault
  - Microsoft.Azure.Services.AppAuthentication
  - Microsoft.Extensions.Configuration.AzureKeyVault
- App Requirements
  - Enable managed identity
- Key Vault Permissions for App Identity
  - Get Secrets
  - List Secrets
- Configuration data is added as a secret to the Key Vault
- Read [this article](https://docs.microsoft.com/en-us/aspnet/core/security/key-vault-configuration?view=aspnetcore-3.1)

---
### Steps
1. Add YOUR key vault URL to configuration file `appsettings.json` (or app configuration in Azure)
```json
{
    "KeyVaultUrl": "https://my-key-vault.vault.azure.net/"
}
```

2. Modify `Program.cs` to use key vault for configuration
```cs
    public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .ConfigureAppConfiguration((context, config) =>
                {
                    if (context.HostingEnvironment.IsProduction())
                    {
                        var builtConfig = config.Build();

                        var azureServiceTokenProvider = new AzureServiceTokenProvider();
                        var keyVaultClient = new KeyVaultClient(
                            new KeyVaultClient.AuthenticationCallback(
                                azureServiceTokenProvider.KeyVaultTokenCallback));

                        config.AddAzureKeyVault(
                                builtConfig["KeyVaultUrl"],
                                keyVaultClient,
                                new DefaultKeyVaultSecretManager());
                    }
                })
                .ConfigureWebHostDefaults(webBuilder =>
                {
                    webBuilder.UseStartup<Startup>();
                });
```

3. Access configuration values in `Startup.cs` as per usual. Typical pattern would be to load sensitive values from key vault and keep rest of the configuration in a config file.
```cs
    services.Configure<ConfigurationData>(o =>
           {
               /* In this case, ServiceConnetionString is pulled from the key vault.
                  Secret name will be ConnectionStrings--ServiceConnetionString,     
                  Note that configuration levels in key vault are separated by -- (two dashes).
                  dotnet user-secrets uses : (colon) to separate them
               */
               o.SensitiveConnectionString = Configuration.GetConnectionString("ServiceConnetionString");

               /* Rest of configuration object is bound through options pattern */
               Configuration.GetSection("ConfigurationData").Bind(o);
           });
```

---
## Storing and accessing encrypted configuration file in Azure Blob Storage

