Since .net core has the ability to run on both Linux and Windows I wanted to explore running code on the Raspberry Pi. I quickly came to the conclusion that I would most likely want to debug my code on the actual Raspberry Pi since I was curious about interacting with the GPIO.

I had to do some digging and found a couple bumps in the road. I wanted to make it easier to get this done and have everything in one place.

Special thanks to Scott Hanselman as his blog post on the same subject was very helpful and I will be repeating some of his steps here. You can read that post here https://www.hanselman.com/blog/remote-debugging-with-vs-code-on-windows-to-a-raspberry-pi-using-net-core-on-arm.


# Image the SD Card

There is a convenient imaging tool on the Raspberry Pi website that will put the image on your SD card. You can find it here https://www.raspberrypi.org/downloads/. Follow the directions on the site to get the OS loaded onto the card.

At the time of this writing I used the August 2020 version of the Raspberry Pi OS which is based on Debian 10.

After you have the image on the SD card place it in your Raspberry Pi and boot it up. Complete the initial setup steps inclusing network configuration and setting up a password.



# Enable SSH on the Raspberry Pi

This is a pretty straightforward process following the instructions here https://www.raspberrypi.org/documentation/remote-access/ssh/

Windows 10 now has an SSH client installed by default. You can use it by going to a terminal window (Windows Terminal is pretty nice https://github.com/microsoft/terminal) and entering `ssh pi@[your pi IP address]`



# Setup Remote Debugging on the Raspberry Pi

In order to debug the code remotely the Visual Studio Remote Debugger will need to be installed on the Pi. Full Disclosure: These steps are identical to Scott's post.

1. Log into your Pi via SSH.
2. Install the debugger.

    `curl -sSSL https://aka.ms/getvsdbgsh | /bin/sh /dev/stdin -v latest -l ~/vsdbg`
3. The debugger needs to run as root, so set the root password.

    `sudo password root`
4. Enable root to establish an SSH connection.

    `sudo nano /etc/ssh/sshd_config`

    Ensure the line below is in the file and not commented.

    `PermitRootLogin yes`
5. Reboot the Pi.
    
    `sudo reboot`


# Install the .net core runtime on the Raspberry Pi

I tried to follow the instructions found here https://docs.microsoft.com/en-us/dotnet/core/install/linux-debian. I was unable to do this as I was getting an error about not being able to find the package with apt-get.

I ended up taking the manual approach and run this command.

    curl -sSL https://download.visualstudio.microsoft.com/download/pr/3f331a87-d2e9-46c1-b7ef-369f8540e966/2e534214982575ee3c79a9ce9f9a4483/dotnet-runtime-3.1.8-linux-arm.tar.gz | tar xvzf /dev/stdin -C ~/dotnet

This command exracted the 3.1.8 version of the .net runtime to the ~/dotnet folder where it can be referenced later.

I was able to find that url by going to the .net core download page https://dotnet.microsoft.com/download/dotnet-core/3.1 and clicking on the link for the ARM32 runtime. This provided me with another page that had a direct link to the zipped file.


# Development Machine Setup

Now we can setup the development machine. I am using Windows so these instructions will follow.

1. Install the .net core SDK (https://dotnet.microsoft.com/download/dotnet-core/3.1 )
2. Install Visual Studio Code
3. Install Putty (https://www.putty.org/)


# Test Remote Debugging

### __Create a new project__
Create a folder on your filesystem called PiTest. On the command line, run the command `dotnet new console`. This will generate a new Hello World console application.


### __Edit the task.json file__

When deploying code to the Raspberry Pi there are 3 things that need to be done.

1. Build
2. Publish
3. Deploy the published application to the Pi.

Use Visual Studio Code to open the PiTest folder. Once In VS Code you will see a .vscode folder in the Explorer window. Open the tasks.json file. This is the json file that I created based on the 3 tasks from above.

``` json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "build",
            "command": "dotnet",
            "type": "process",
            "args": [
                "build",
                "${workspaceFolder}/PiTest.csproj",
                "/property:GenerateFullPaths=true",
                "/consoleloggerparameters:NoSummary"
            ],
            "problemMatcher": "$msCompile"
        },
        {
            "label": "dotnetPublish",
            "command": "dotnet",
            "type": "process",
            "presentation": {
                "reveal": "always",
                "panel": "shared",
                "showReuseMessage": false,
                "echo": false,
                "clear": true
            },
            "options": {
                "cwd": "${workspaceFolder}"
            },
            "args": [
                "publish",
                "${workspaceFolder}/PiTest.csproj",
                "-r",
                "linux-arm"
            ]
        },
        {
            "label": "pscp",
            "type": "shell",
            "options": {
                "cwd":"${workspaceFolder}"
            },
            "windows":{
                "command": "pscp",
                "args": [
                    "-pw",
                    "${env:pi_pwd}",
                    "-v",
                    "-r",
                    "${cwd}\\bin\\Debug\\netcoreapp3.1\\linux-arm\\publish\\*",
                    "pi@192.168.1.89:/home/pi/Desktop/rpitest"
                ]
            },
            "presentation": {
                "reveal": "always",
                "panel": "shared",
                "showReuseMessage": false,
                "echo": false
            },
        },
        {
            "label": "PublishAndDeploy",
            "type": "shell",
            "windows":{
                "command": "echo",
                "args": ["'Done'"]
            },
            "presentation": {
                "reveal": "always",
                "panel": "shared",
                "showReuseMessage": false,
                "echo": false
            },
            "dependsOrder": "sequence",
            "dependsOn":["dotnetPublish", "pscp"]
        }
    ]
}

```

This file is using compound tasks (https://code.visualstudio.com/docs/editor/tasks#_compound-tasks). Compound Tasks are a fancy way to chain tasks together.

The 'build' task was provided out of the box.

The 'PubishAndDeploy' task has dependencies on 'dotnetPublish' and 'pscp'. The dependent tasks are called first and since the dependsOrder is set to 'sequence' they will be called in the order listed.

The 'dotnetPublish' task called the dotnet command to publish the project. This will include building the project before it is published.

The 'pscp' task calls the pscp shell command to copy the published files to the Pi.

Since a best practice is to always use source control you would want to source control this file. One thing to note is that you will need the password for the pi user in this file in order copy the files. You will notice that there is a placeholder that looks like this '${env:pi_pwd}'. This is the syntax for getting an environment variable. I wanted to be able to commit the tasks.json file into source control and not have the password so I opted to use and environment variable. Of course, that environment variable will need to populated with the correct password for this to work.

That should give us everything we need to build, publish and deploy as mentioned above.

Press Ctrl + Shift + P to open the command pallette. Then select 'Tasks: Run Task'. Then select 'PublishAndDeploy. This should execute the tasks and deploy the code to the Pi.

### __Edit the launch.json File__
Now that we have the project building and deploying what about debugging? Well we will have to execute the application and then attach to it with the debugger. It would be nice to be able to have this automated when when you clicked the play button in Run window in VS Code.

For this we will need to edit the launch.json file. This is the final result that I came up with.

``` json
{
   // Use IntelliSense to find out which attributes exist for C# debugging
   // Use hover for the description of the existing attributes
   // For further information visit https://github.com/OmniSharp/omnisharp-vscode/blob/master/debugger-launchjson.md
   "version": "0.2.0",
   "configurations": [
        {
            "name": ".NET Core Launch (console)",
            "type": "coreclr",
            "request": "launch",
            "preLaunchTask": "build",
            // If you have changed target frameworks, make sure to update the program path.
            "program": "${workspaceFolder}/bin/Debug/netcoreapp3.1/PiTest.dll",
            "args": [],
            "cwd": "${workspaceFolder}",
            // For more information about the 'console' field, see https://aka.ms/VSCode-CS-LaunchJson-Console
            "console": "internalConsole",
            "stopAtEntry": false
        },
        {
            "name": ".NET Core Launch (remote console)",
            "type": "coreclr",
            "request": "launch",
            "preLaunchTask": "PublishAndDeploy",
            // If you have changed target frameworks, make sure to update the program path.
            "program": "/home/pi/dotnet/dotnet",
            "args": ["/home/pi/Desktop/rpitest/PiTest.dll"],
            "cwd": "/home/pi/Desktop/rpitest",
            // For more information about the 'console' field, see https://aka.ms/VSCode-CS-LaunchJson-Console
            "console": "internalConsole",
            "stopAtEntry": false,
            "pipeTransport": {
                "pipeCwd": "${workspaceFolder}",
                "pipeProgram": "plink.exe",
                "pipeArgs": [
                    "-pw",
                    "${env:root_pwd}",
                    "root@192.168.1.89"
                ],
                "debuggerPath": "/home/pi/vsdbg/vsdbg"
            }
        },
        {
            "name": ".NET Core Attach",
            "type": "coreclr",
            "request": "attach",
            "processId": "${command:pickProcess}"
        }
    ]
}
```

The '.NET Core Launch (console)' and '.NET Core Attach' configurations are provided out of the box. Copy and edit the '.NET Core Launch (Console) configuration to look like the one above.

The first thing to note is that the paths are relative to the Pi and not to the development machine. The second notable item is that the root user is being used to debug and the password will be needed for that user. I again used an environment variable to store the password so that it would not get commited to source control.

At this point you should be able to place a breakpoint in the `Main` method of the application, go to the Run window, choose '.NET Core Lauch (remote console)' and press the play button. The breakpoint placed in the `Main` method should be hit. The console applications output should be displayed in the DEBUG CONSOLE.