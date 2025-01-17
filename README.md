# WIP, not ready for public consumption yet

# Unreal Dedicated Servers with Steam sample application

## Requirements

- Ability to compile the engine
- Two Windows PCs
- Two Steam accounts

This sample demonstrates how to get dedicated servers working with Steam on UE 5.3.

This sample uses the popular Advanced Sessions plugin for creating and joining sessions, since it is a popular plugin for indie devs, but it is not a requirement for dedicated servers.

## Important: Firewall

The PC hosting the server MUST have its ports open. UDP port 27015 is used with the Steam master server, and port 7777 is used to actually connect to the game server.

How this is configured depends on your networking and router setup. Also make sure your software firewall (Windows Defender etc.) is configured to allow incoming traffic on those ports.

![ports](Docs/ports.png)

## Downloading the engine

Using dedicated servers with Steam WILL require a custom Engine build. This is due to some preprocessor defines (UE_PROJECT_STEAMPRODUCTNAME etc.) that cannot be changed without a full engine rebuild.
In addition, it is highly useful for servers to enable logging (UE_LOG etc) in Shipping builds, and this also requires a custom build.

```basic summary for updating the engine here```

Remember to right-click the .uproject and switch the engine version to the one you have downloaded!

## Building the Client application

Let's take a look at DediServerSample.Target.cs. This target will build the game client.

```
public DediServerSampleTarget(TargetInfo Target) : base(Target)
{
    Type = TargetType.Game;
    DefaultBuildSettings = BuildSettingsVersion.V4;
    IncludeOrderVersion = EngineIncludeOrderVersion.Unreal5_3;
    DediServerSampleTarget.ApplySharedTargetSettings(this);
    ExtraModuleNames.Add("DediServerSample");
    DisablePlugins.Add("OpenImageDenoise");
}

internal static void ApplySharedTargetSettings(TargetRules Target)
{
    Target.bUseLoggingInShipping = true;
    Target.GlobalDefinitions.Add("UE_PROJECT_STEAMSHIPPINGID=480");
    Target.GlobalDefinitions.Add("UE_PROJECT_STEAMPRODUCTNAME=\"spacewar\"");
    Target.GlobalDefinitions.Add("UE_PROJECT_STEAMGAMEDIR=\"spacewar\"");
    Target.GlobalDefinitions.Add("UE_PROJECT_STEAMGAMEDESC=\"Spacewar\"");
}
```

The four UE_PROJECT_... definitions need to be changed for your own application. This sample uses Valve's 'spacewar' test application ID, which is a common way of developing Steam games with Unreal before acquiring your proper application ID.

Note if you don't want to use the 'logging in shipping' flag, feel free to remove it, it is not required for this sample to work.

There's a 'one-click' build script included in this repository, in Tools/Build/build_development_client.cmd. It looks like this:

```
@echo "Building Development"
@set PROJECT=%~dp0../../DediServerSample.uproject
@set ENGINEPATH=D:/git/UnrealEngine-Angelscript
@set OUTPUT=%~dp0../Steam/content
"%ENGINEPATH%/Engine/Build/BatchFiles/RunUAT" BuildCookRun^
 -project="%PROJECT%"^
 -targetplatform=Win64^
 -noserver^
 -clientconfig=Development^
 -serverconfig=Development^
 -build^
 -cook^
 -pak^
 -stage^
 -nop4^
 -utf8output^
 -stagingdirectory="%OUTPUT%"
@pause
```

Change the ENGINEPATH to point to your actual engine location (no trailing slash), and of course if you use this in your own project change the .uproject path as well.
Running this script will build the Engine, editor and game in Development configuration, cook the content and finally package it under Tools/Steam/Content/Windows (handy for later steam uploading).

## Building the Server application

DediServerSampleServer.Target.cs does not need to be modified, as it sets up the Steam defines with ```DediServerSampleTarget.ApplySharedTargetSettings(this);```.

```
public DediServerSampleServerTarget(TargetInfo Target) : base(Target)
{
    Type = TargetType.Server;
    DefaultBuildSettings = BuildSettingsVersion.V4;
    IncludeOrderVersion = EngineIncludeOrderVersion.Unreal5_3;
    DediServerSampleTarget.ApplySharedTargetSettings(this);
    ExtraModuleNames.Add("DediServerSample");
    DisablePlugins.Add("OpenImageDenoise");
}
```

As with the client, there is a simple build script provided under Tools/Build/build_development.server. Edit the engine/uproject paths and run the script. The final build should be placed under Tools/Steam/content/WindowsServer

## Running the server

Launch the server from the command line, e.g.

```D:\UnrealProjects\DediServerSample\Tools\Steam\content>WindowsServer\DediServerSampleServer.exe -log```

This way you will get the log window to easily see any status messages. After a successfull launch, the last messages in the log should be:

```
[ 82]LogBlueprintUserMessages: [BP_GameInstance_C_2147482578] Server created
LogBlueprintUserMessages: [BP_GameInstance_C_2147482578] Started session
```

If there's ANY kind of issue with initializing Steam, you should get a fairly obvious message about it. For example, if Steam cannot launch due to the game client also running on the same PC, you would see:

```
LogSteamShared: Warning: Steam Dedicated Server API failed to initialize.
LogOnline: STEAM: [AppId: 480] Game Server API initialized 0
LogOnline: Warning: STEAM: Failed to initialize Steam, this could be due to a Steam server and client running on the same machine. Try running with -NOSTEAM on the cmdline to disable.
LogOnline: Display: STEAM: OnlineSubsystemSteam::Shutdown()
LogOnline: Warning: STEAM: Steam API failed to initialize!
```

## Verifying the Server is visible in Steam

In Steam, go to View->Game Servers and look for "Spacewar" servers. The sample application server will show up like this:

![steam browser](Docs/steambrowser.png)

The sample server shows up with DummyMapName and 57 open player slots. The rest are other devs testing their own games.

If it doesn't show up:
1. Check the server log that no errors or warnings occurred during server launch
2. Check your firewall setup.

## Troubleshooting: Testing without Steam

This can help to rule out firewall issues.

- Launch the server with -nosteam
- Launch the client with -nosteam
- In client, open console and type "open 127.0.0.1" (if the server is running on the same PC)

The client should successfully connect to the server. If it doesn't, check your firewall setup.

## Troubleshooting: Taking Unreal out of the picture

Explain how to use the Spacewar sample to troubleshoot server browsing

## FAQ: Testing the join flow in Editor

You can't run the game server and game client at the same time connected to Steam. You can however run the client in Standalone mode (Steam should be running and logged on in the background), and run your prebuilt server on another PC and connect to it. This makes it easier to develop and test your server browser/join flow.

If you don't run the client in Standalone, it won't launch with Steam but will use the 'null online subsystem'.

## FAQ: steam_appid.txt

Deploying a steam_appid.txt with the server binary is necessary only if the executable is not allowed to write to its directory. Normally the .txt file will be created when the application launches.

## FAQ: The server cannot find the Steam DLLs!

## Bonus: Uploading your game to Steam

Example Steam upload scripts (.vdf and .cmd files) are found under Tools/Steam/. First you should download Steamcmd.exe from https://steamcdn-a.akamaihd.net/client/installer/steamcmd.zip and place the .exe in Tools/Steam/Builder. Then you'll need to modify the .vdf files to use your own app IDs and file names, but once they're set up you should be able to deploy new builds to Steam simply by running the Build scripts and then the Upload .cmd.

## Bonus: Running a server on Steam Deck for testing

Explain how to launch a serer on the Deck

## License

This sample is licensed under MIT.

```
Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the “Software”), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
```