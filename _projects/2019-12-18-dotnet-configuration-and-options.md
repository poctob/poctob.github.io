---
layout: project
title: Dotnet core configuration and options pattern
description: Simple recipie of options pattern common use
summary: 
category: Dotnet core recipes
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

## Using Azure Key Vault to manage and access sensitive configuration values

## Storing and accessing encrypted configuration file in Azure Blob Storage

