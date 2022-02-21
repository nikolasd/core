# .NET 6.0 Known Issues

You may encounter the following known issues, which may include workarounds, mitigations or expected resolution timeframes.

## Failure to install the .NET 6.0.1 update via Microsoft Update

#### Summary
There have been limited reports of a failure to install the .NET 6.0.1 update via Microsoft Update, the update fails with an error code 0x80070643.

.NET 6.0 can be updated to 6.0.1 via MU and .NET 6.0.1 is also included in the Visual Studio 17.0.3 update. Both options carry the .NET Core Runtime and ASP.NET Core runtime version 6.0.1 and the .NET 6 SDK version 6.0.101. When these are installed, applications will by default roll forward to using the latest runtime patch version automatically. See [framework dependent app runtime roll forward](https://docs.microsoft.com/en-us/dotnet/core/versions/selection#framework-dependent-apps-roll-forward) for more information about this behavior.

Therefore, installing either the 6.0.1 update via MU or the VS 17.0.3 update will secure the machine for the vulnerability described in [CVE-2021-43877](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2021-43877).


#### Root Cause
The optional workload manifest MSIs in the SDK populate the Language column in the Upgrade table. The INSTALLEDLANGUAGE property cannot be queried under the USERUNMANAGED context, it can only be queried under MSIINSTALLCONTEXT_MACHINE context. Due to an error the .NET 6.0.101 SDK Wix bundle sets the installer context incorrectly to USERUNMANAGED when running under the LOCAL\SYSTEM account. This causes the engine to continue and execute an older copy of the MSI instead of skipping it, which in turn triggers a launch condition to block the downgrade and the subsequent error causes the bundle to fail, resulting in the MU update failure.


#### Workaround
Running the 6.0.101 SDK bundle (without using MU) results in the context changing to MSIINSTALLCONTEXT_MACHINE, this allows the API call to query the INSTALLEDLANGUAGE to complete and the SDK Wix bundle install succeeds.

Therefore a workaround for this issue is to install the 6.0.101 SDK bundle manually by downloading it from the [.NET download site](https://dotnet.microsoft.com/en-us/download/dotnet/6.0). Once this is successfully installed scanning MU again will result in clearing the previous error. 

As described previously the computer can be secured by installing the VS 17.0.3 update, even if the MU update results in a failure so the MU failure is not a critical factor from a security perspective. Therefore for the case where we expect the VS update to offer and secure the computer we will be making a change to not offer the MU update to those computers to avoid the MU failure. For the case where .NET 6 was installed as a standalone version and VS is not expected to patch the computer we will continue to offer the 6.0.1 update via MU. 


## .NET SDK

.NET 6 is supported with Visual Studio 2022 and MSBuild 17.  It is not supported with Visual Studio 2019 and MSBuild 16.

If you build .NET 6 projects with MSBuild 16.11, for example, you will see the following error:

`warning NETSDK1182: Targeting .NET 6.0 in Visual Studio 2019 is not supported`

You can use the .net 6 SDK to target downlevel runtimes in 16.11.

#### 1. dotnet test x64 emulation on arm64 support
While a lot of work has been done to support arm64 emulation of x64 processes in .net 6, there are some remaining [work](https://github.com/dotnet/sdk/issues/21686) to be done in 6.0.2xx. The most impactful remaining item is `dotnet test` support.

`dotnet test --arch x64` on an arm64 machine will not work as it will not find the correct test host.  To test x64 components on an arm64 machine, you will have to install the x64 SDK and configure your `DOTNET_ROOT` and `PATH` to use the x64 version of dotnet.

#### 2. Upgrade of Visual Studio or .NET SDK from earlier builds can result in a bad `PATH` configuration on Windows
When upgrading Visual Studio to preview 5 or the .NET SDK to RC2 from an earlier build, the installer will uninstall the prior version of the .NET Host (dotnet.exe) and then install a new version. This results in the path to the x64 copy of dotnet being removed from the `PATH` then added back. If you have the x86 .NET Host installed, it will end up ahead of the x64 one and will be picked up first. 

In this case you may find that Visual Studio is unable to create projects, or commands like `dotnet new` fail with a message like this:
```
Could not execute because the application was not found or a compatible .NET SDK is not installed.
```

To confirm, run `dotnet --info` and you'll see (x86) paths for all of the found .NET runtimes and .NET SDKs installed. 

To fix this, edit your `PATH` environment variable to either remove the `c:\Program Files (x86)\dotnet` entry or move it after the entry for `c:\Program Files\dotnet`. Now reopen your console window.
   
## ASP.NET Core

### SPA template issues with Individual authentication when running in development

The first time SPA apps are run, the authority for the spa proxy might be incorrectly cached which results in the JWT bearer being rejected due to Invalid issuer. The workaround is to just restart the SPA app and the issue will be resolved. If restarting doesn't resolve the problem, another workaround is to specify the authority for your app in Program.cs: `builder.Services.Configure<JwtBearerOptions>("IdentityServerJwtBearer", o => o.Authority = "https://localhost:44416");` where 44416 is the port for the spa proxy.

When using localdb (default when creating projects in VS), the normal database apply migrations error page will not be displayed correctly due to the spa proxy. This will result in errors when going to the fetch data page. Apply the migrations via 'dotnet ef database update' to create the database.
