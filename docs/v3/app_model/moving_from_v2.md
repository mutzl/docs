---
id: app_model_moving_from_v2
title: Moving from NetDaemon version 2
---

If you are using HassClient today and want to migrate to V3 you have two options:

   1. Manually copy your existing apps over to a new copy of the V3 app template. *This is the recommended approach.* [See how to get the template here](../started/development.md#use-dotnet-cli-tool)
   2. Manually update your project to support the new V3 packages. This is recommended if you have a more complex NetDaemon project - see the [section below](#upgrading-an-existing-apps-project-to-v3) for more details.

:::warning

**Do not mix the models in the same project! If you are upgrading app by app and have both V2 and V3 apps for a time please use two different deployments/add-ons and locations for your V2 and V3 apps!**

:::

### Main changes

Here is a summary of the main changes between the versions:

- New namespaces `NetDaemon.AppModel` and `NetDaemon.Runtime`. If you are using HasClient today you should just replace any `NetDaemon.xyz` with `NetDaemon.AppModel`. We recommend to check out [how to get the template here](../started/development.md#use-dotnet-cli-tool).
- Configuration using YAML is changed. Configuration is now only available as injected `IAppConfig<Class>` in th constructor. To migrate, copy public properties to your own config class and change the YAML using that class name instead of app name. The class property on YAML can be removed.
- Base class based on `NetDaemonRxApp` is not supported at all. You will need to migrate your applications to HassModel before upgrading to NetDaemon 3.

### Upgrading an existing app project to v3

There are five main considerations when upgrading a new project to use V3 _(before you begin taking advantage of the new functionality!_):

1. Update `program.cs` to use the new application startup and configuration
1. Update your nuget packages
1. Change `GlobalUsings.cs` to refer to the new namespace
1. Update `appsettings.json` to reflect a change in config
1. Update your locally installed code generator

After you have tested your project locally you will need to update your NetDaemon install on Home Assistant to v3.

### 1. Update program.cs

Refer to your project's `program.cs` and check the main application startup block. You will likely have a section that looks like this:

```csharp
using Microsoft.Extensions.Hosting;
using NetDaemon;

try
{
    await Host.CreateDefaultBuilder(args)
        .UseDefaultNetDaemonLogging()
        .UseNetDaemon()
        .Build()
        .RunAsync();
    // etc...
```

Update this to reflect the makeup of the new V3 app template's [program.cs](https://github.com/net-daemon/netdaemon-app-template/blob/v3/DefaultDeploy/program.cs) - before you do this, check whether you are using a custom logger or any other custom configuration, and copy this out of the way. You can copy it back again after you have updated to the V3 startup.

:::warning

Please check the up-to-date [version here](https://github.com/net-daemon/netdaemon-app-template/blob/v3/DefaultDeploy/program.cs) rather than copy/pasting from this page. The example below is just to give you an idea of what the new code looks like but the app template may have been updated since this documentation was written.

:::

When finished, your `program.cs` should look similar to this:

```csharp
using System;
using Microsoft.Extensions.Hosting;
using NetDaemon.Runtime;
using NetDaemon.AppModel;
using NetDaemon.Extensions.Logging;
using NetDaemon.Extensions.Scheduler;
using NetDaemon.Extensions.Tts;
using System.Reflection;

#pragma warning disable CA1812

try
{
    await Host.CreateDefaultBuilder(args)
        .UseNetDaemonAppSettings()
        .UseNetDaemonDefaultLogging()
        .UseNetDaemonRuntime()
        .UseNetDaemonTextToSpeech()
        .ConfigureServices((_, services) =>
            services
                .AddAppsFromAssembly(Assembly.GetExecutingAssembly())
                .AddNetDaemonStateManager()
                .AddNetDaemonScheduler()
        )
        .Build()
        .RunAsync()
        .ConfigureAwait(false);
    // etc...
```

### 2. Update your nuget packages

Remove the following packages:

- JoySoftware.NetDaemon.App
- JoySoftware.NetDaemon.Client

Add the following packages:

- JoySoftware.NetDaemon.AppModel
- JoySoftware.NetDaemon.DaemonRunner
- JoySoftware.NetDaemon.Extensions.Tts
- JoySoftware.NetDaemon.Runtime

Finally, ensure that all packages are updated to the latest versions.

### 3. Update global usings

In `apps/GlobalUsings.cs` replace `NetDaemon.Common` with `NetDaemon.AppModel`, and `NetDaemon.HassModel.Common` with `NetDaemon.HassModel`:

```csharp
// Common usings for NetDaemon apps
global using System;
global using System.Reactive.Linq;
global using Microsoft.Extensions.Logging;
global using NetDaemon.AppModel;   // <--- changed
global using NetDaemon.HassModel;  // <--- changed
```

### 4. Update appsettings.json

There is a breaking change in your `appsettings.json` file - in the `NetDaemon` section rename the `AppSource` key to `ApplicationConfigurationFolder`. Note also that the default application source folder has changed from `"."` to `"./apps"`.

V2 variant:

```json
  // excerpt
    "NetDaemon": {
    "AppSource": "."
  },
  // ...
```

V3 variant:

```json
  "NetDaemon": {
    "ApplicationConfigurationFolder": "./apps"
  },
  // ...
```

### 5. Update the code generator

As some of the namespaces have changed you should update your locally installed version of the [Code generator](/v3/hass_model/hass_model_codegen.md) by running the following command:

```bash
dotnet tool install -g JoySoftware.NetDaemon.HassModel.CodeGen
```

...and then re-run the generator to create an updated version of `HomeAssistantGenerated.cs`

### Update the version of NetDaemon in Home Assistant

If you still have V2 of NetDaemon installed inside Home Assistant ("HA") then that will not be able to run your V3 apps. You should remove the V2 install from HA first, then install the V3 version.

After you have done so remember to check the configuration (the default config path changed from `/config/netdaemon` to `/config/netdaemon3`) and re-deploy your apps project to it, then restart NetDaemon and check the logs.

You can find the [installation instructions here](v3/started/installation.md)
