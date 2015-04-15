---
layout: page
title: Cloud Deployment
---
{% include JB/setup %}

## Deploying Orleans to Windows Azure
This walkthrough shows the steps required to deploy the sample created in the  [Front Ends for Orleans Services](Front-Ends-for-Orleans-Services) to Windows Azure Cloud Services.

 In Azure, one or more worker roles will be used to host the Orleans silos, and an Azure web role will act as the presentation layer for the application and client to the application grains running in the Orleans silos. 

 The same grain interfaces and implementation can run on both Windows Server and Windows Azure, so no special considerations are needed in order to be able to run your application in a Windows Azure hosting environment.

 The  [Azure Web Sample] (https://orleans.codeplex.com/wikipage?title=Azure%20Web%20Sample&referringTitle=Cloud%20Deployment) sample app in the Orleans SDK provides a working example of how to run a web application with supporting Orleans silo cluster backend in an Azure hosted service, and the details are described below.

## Pre-requisites
The following prerequisites are required:
* Visual Studio 2013 
* Orleans SDK 
* [Windows Azure SDK v2.4](http://azure.microsoft.com/en-us/downloads/)

 For more info on installing and working with Windows Azure in general, see the Microsoft Developer Network (MSDN) web site: http://msdn.microsoft.com/en-us/windowsazure/ 

## Create Azure Worker Role for Orleans Silos 
Right click on your solution, and select 'Add | New Project...'.

 Choose the 'Windows Azure Cloud Service' project template:

![](http://download-codeplex.sec.s-msft.com/Download?ProjectName=orleans&DownloadId=815101)

 When prompted, select a new Worker Role to add to the project. This will bed used to host the Orleans Silos:

![](http://download-codeplex.sec.s-msft.com/Download?ProjectName=orleans&DownloadId=815102)

## Add project references for Orleans Silo binaries 
Add references to the OrleansAzureSilos project for the required Orleans server library files. Copies of these files can be found in the .\Binaries\OrleansServer folder under the Orleans SDK.
* Orleans.dll 
* OrleansAzureUtils.dll 
* OrleansRuntime.dll 
* OrleansProviders.dll (Only required if you your grains use the Declarative Persistence functionality.)

 Note: All of these references MUST have Copy Local = `True` settings to ensure the necessary library DLLs get copied into the OrleansAzureSilos project output directory. 

![](http://download-codeplex.sec.s-msft.com/Download?ProjectName=orleans&DownloadId=815103)

## Configure Azure Worker Role for Orleans Silos 
The Worker Role initialization class is a normal Azure worker role - it needs to inherit from the usual `Microsoft.WindowsAzure.ServiceRuntime.RoleEntryPoint` base class. 

 The worker role initialization class needs to create an instance of `Orleans.Host.Azure.OrleansAzureSilo` class, and call the appropriate Start/Run/Stop functions in the appropriate places:

``` csharp
public class WorkerRole : RoleEntryPoint 
{ 
    Orleans.Host.Azure.OrleansAzureSilo silo; 

    public override bool OnStart() 
    { 
        // Do other silo initialization – for example: Azure diagnostics, etc 

        silo = new OrleansAzureSilo(); 

        return silo.Start( 
            RoleEnvironment.DeploymentId, RoleEnvironment.CurrentRoleInstance); 
    } 
    public override void OnStop() { silo.Stop(); } 
    public override void Run()    { silo.Run(); } 
} 
```

 Then, in the ServiceDefinition.csdef file for this role, add some required configuration items used by the Orleans Azure hosting library to the WorkerRole configuration:
* Add a ConfigurationSettings declaration named 'DataConnectionString' This is the Azure storage location where Orleans Azure hosting library will place / look for its silo instance table. 
* Add a LocalStorage declaration named 'LocalStoreDirectory' This is the directory that will be used for any local cache directories. 
* Add an InternalEndpoint declaration for a TCP endpoint named 'OrleansSiloEndpoint' 
* Add an InternalEndpoint declaration for a TCP endpoint named 'OrleansProxyEndpoint' 


    <ServiceDefinition ...>
    <WorkerRole name="OrleansAzureSilos" ...>
      ... 
      <ConfigurationSettings>
        <Setting name="DataConnectionString" />
      </ConfigurationSettings> 
      <Endpoints> 
        <InternalEndpoint name="OrleansSiloEndpoint" protocol="tcp" port="11111" /> 
        <InternalEndpoint name="OrleansProxyEndpoint" protocol="tcp" port="30000" /> 
      </Endpoints> 
      ... 
    </WorkerRole> 
    ... 
    </ServiceDefinition> 


 In the ServiceConfiguration.cscfg file for this role, add some required configuration items used by the Orleans Azure hosting library:

 Add a ConfigurationSettings definition for 'DataConnectionString'

 This will be a normal Azure storage connection – either for the development storage emulator (only valid if running locally), or a full Azure storage account connection string for cloud-storage.

 In general, this connection string is likely to use the same configuration values as the `Microsoft.WindowsAzure.Plugins.Diagnostics.ConnectionString` diagnostics connection string setting, but is not required to.

http://msdn.microsoft.com/en-us/library/ee758697.aspx

 For example, to use local Developer Storage emulator (for local testing only)


    <ConfigurationSettings> 
       <Setting name="DataConnectionString" value="UseDevelopmentStorage=true" /> 
    </ConfigurationSettings> 


 Or using an Azure cloud storage account:


    <ConfigurationSettings> 
       <Setting name="DataConnectionString" value="DefaultEndpointsProtocol=https;AccountName=MyAccount;AccountKey=MyKey" /> 
    </ConfigurationSettings> 


## Changing DataConnectionString
The "DataConnectionString" setting in ServiceDefinition.csdef is the default name use by the Orleans silo for the Azure table account connection string that will be used for Orleans system tables such as OrleanSiloInstances. 

 If you wish to use a different setting name for this value, then in the silo role initialization code set the property OrleansAzureSilo.DataConnectionConfigurationSettingName before the call to OrleansAzureSilo.Start

Add your Orleans silo config file to Azure Worker Role for Orleans Silos 
Add an OrleansConfiguration.xml file into your OrleansAzureSilos worker role project, along with any supporting libraries those grains need. The networking configuration information in OrleansConfiguration.xml will be overridden by values from the Azure role environment / configuration.


Note: You MUST ensure this config file get copied into the OrleansAzureSilos project output directory, to ensure they get picked up by the Azure packaging tools. 
![](http://download-codeplex.sec.s-msft.com/Download?ProjectName=orleans&DownloadId=815104)

## Add your grain binaries to Azure Worker Role for Orleans Silos 
Add the grain interfaces DLL and implementation classes DLL for the grains to he hosted in the Azure silo into the OrleansAzureSilos project, along with any supporting libraries those grains need.


Note: You MUST ensure that all the referenced binaries are copied into the OrleansAzureSilos project output directory, to ensure they get picked up by the Azure packaging tools.
![](http://download-codeplex.sec.s-msft.com/Download?ProjectName=orleans&DownloadId=815105)

## Running Orleans Client as Azure Web Role
The user interface / presentation layer for your application will usually run as a Web Role in Azure.

 The Orleans.Host.Azure.Client.OrleansAzureClient utility class is the main mechanism for bootstrapping connection to the Orleans silo worker roles from an Azure Web Role. A few additional configuration steps are needed to make the OrleansAzureClient utility class work – see below for details.

**Create Azure Web Role for Orleans Client **
Using the Azure Tools for Visual Studio, create a normal Azure web role project.

 Any type of web role can be used as an Orleans client, and there are no specific naming requirements or conventions for this project.

 Add a web role to the solution:

![](http://download-codeplex.sec.s-msft.com/Download?ProjectName=orleans&DownloadId=815106)

Add project references for Orleans Client binaries 
Add references to the web role project for the required Orleans client library files. Copies of these files can be found in either  SDK-ROOT\Samples\References or  SDK-ROOT\Binaries\OrleansClient directories.
* Orleans.dll 
* OrleansAzureUtils.dll


Note: All of these references MUST have Copy Local = True settings to ensure the necessary library DLLs get copied into the web role project output directory. 
08.png

**Configure Azure Web Role to be an Orleans Client **
In the ServiceDefinition.csdef file for this web role, add some required configuration items used by the Orleans Azure hosting library:
* Add a ConfigurationSettings declaration named 'DataConnectionString'. 
This is the Azure storage location where Orleans Azure hosting library will place / look for its silo instance table. 
* In addition, a http/s InputEndpoint will also need to be declared, just as for any other Azure web role config. 


    <ServiceDefinition ...> 
    <WebRole name="MyWebRole" ...> 
      ... 
      <ConfigurationSettings: 
        <Setting name="DataConnectionString" /: 
      </ConfigurationSettings> 
      <Sites> 
        <Site name="Web"> 
          <Bindings> 
            <Binding name="Endpoint1" endpointName="Endpoint1" /> 
          </Bindings> 
        </Site> 
      </Sites> 
     <Endpoints> 
       <InputEndpoint name="Endpoint1" protocol="http" port="80" /> 
     </Endpoints> 
      ... 
    </WebRole> 
    ... 
    </ServiceDefinition> 


 In the ServiceConfiguration.cscfg file for this role, add some required configuration items used by the Orleans Azure hosting library:

 Add a ConfigurationSettings definition for 'DataConnectionString'.
 This will be a normal Azure storage connection – either for the development storage emulator (if running locally), or a full Azure storage account connection string for cloud-storage.

 This setting MUST match the DataConnectionString value used by the OrleansSiloWorker role in order for the client to discover and bootstrap the connection to the Orleans silos.

 For example, to use local Developer Storage emulator (for local testing only)


    <ServiceConfiguration ...>
    <Role name="OrleansWebClient" ...> 
      ... 
      <ConfigurationSettings> 
        <Setting name="DataConnectionString" value="UseDevelopmentStorage=true" /> 
      </ConfigurationSettings> 
      ... 
    </Role> 
    ... 
    </ServiceConfiguration> 


 Or using an Azure cloud storage account:


    <ServiceConfiguration ...> 
    <Role name="OrleansWebClient" ...> 
      ... 
      <ConfigurationSettings> 
        <Setting name="DataConnectionString>    value="DefaultEndpointsProtocol=https;AccountName=MyAccount;AccountKey=MyKey" /> 
      </ConfigurationSettings> 
      ... 
    </Role> 
    ... 
    </ServiceConfiguration> 


Add your Orleans client config file to Azure Web Role for Orleans Client 
Add a ClientConfiguration.xml file into this project. The networking configuration information in ClientConfiguration.xml will be overridden by values from the Azure role environment / configuration.


Note: You MUST ensure this config file get copied into the web role project output directory, to ensure they get picked up by the Azure packaging tools.
09.png

**Add your grain interface binaries to Azure Web Role for Orleans Client **
Add the grain interfaces DLL for the application grains into this web role project. 

 Access to the DLL containing the grain implementation classes should not be required by the client web role. 


Note: You MUST ensure that all the referenced binaries for grain interfaces and the generated proxy / factory libraries are copied into the web role project output directory, to ensure they get picked up by the Azure packaging tools.
![](http://download-codeplex.sec.s-msft.com/Download?ProjectName=orleans&DownloadId=815108)

 The grain implementation DLLs should not be required by the client and so should not be referenced by the web role.

## Initialize Client Connection to Orleans Silos 
It is recommended to bootstrap and initialize the client connection to the Orleans silo worker roles, to ensure a connection is set up before use – either in the Page_Load method for each .aspx page, or in Global.asax initialization methods. 

``` csharp
protected void Page_Load(object sender, EventArgs e) 
{ 
    if (Page.IsPostBack) 
    { 
        if (!OrleansAzureClient.IsInitialized) 
        { 
            OrleansAzureClient.Initialize(); 
        } 
    } 
} 
```

 Repeated calls to OrleansAzureClient.Initialize()  will return and do nothing if the Orleans client connection is already set up.

 An additional variant of OrleansAzureClient.Initialize(System.IO.FileInfo) allows a base client config file location to be specified explicitly. The internal endpoint addresses of the Orleans silo nodes will still be determined dynamically from the Orleans Silo instance table each silo node registers with. 

 See the Orleans API docs for details of the various Initialize methods available.

## Deploying to Azure
The normal Azure deployment tools are used to deploy the application to Windows Azure – either into the local Azure Compute Emulator for local development / test (Press F5 to run), or into the Azure cloud hosting environment right click on the Cloud project and select 'Publish':

![](http://download-codeplex.sec.s-msft.com/Download?ProjectName=orleans&DownloadId=815110)

## Next
We'll write our own storage provider, to save grain state to alternative to file instead of Windows Azure:

[Custom Storage Providers](Custom-Storage-Providers)
