---------------------
===Getting Started===
---------------------
InstallLion.pkg requires some assembly. At the very least, you must provide your own copy of InstallESD.dmg. This can be found inside the "Install Mac OS X Lion.app" Mac App Store download inside the Contents/SharedSupport subfolder.

Copy InstallESD.dmg into InstallLion.pkg/Contents/Resources/.

An example:

sudo cp /Applications/Install\ Mac\ OS\ X\ Lion.app/Contents/SharedSupport/InstallESD.dmg ./InstallLion.pkg/Contents/Resources/

That's all you need to do for a basic uncustomized installation -- InstallLion.pkg can now be used to do an unattended install of Mac OS X Lion.


------------------
===How it works===
------------------
InstallLion.pkg is a 'payload-free' package -- that is, it does not install anything from the traditional Archive.pax.gz payload found in most packages. Instead, the real work is done as a package postflight script located at InstallLion.pkg/Contents/Resources/postflight.

The postflight script performs the actions that the GUI "Install Mac OS X Lion" application does when you choose to install Lion on the current startup disk.

Those actions are: 
  1) Create a "Mac OS X Install Data" directory at the root of the target volume.
  2) Mount the InstallESD.dmg disk image.
  3) Copy the kernelcache and boot.efi files from the disk image to the "Mac OS X Install Data" directory.
  4) Unmounts (ejects) the InstallESD.dmg disk image.
  5) If the InstallLion.pkg is on the same volume as the target volume, create a hard link to the InstallESD.dmg disk image in "Mac OS X Install Data", otherwise copy the InstallESD.dmg disk image to that directory.
  6) Create a com.apple.Boot.plist file in the "Mac OS X Install Data" directory which tells the kernel how to mount the disk image to use for booting.
  7) Creates a minstallconfig.xml file, which tells the OS X Installer what to install and to which volume to install it. It also provides a path to a MacOSXInstaller.choiceChanges file if one has been included in the package.
  8) Creates an index.sproduct file and an OSInstallAttr.plist in the "Mac OS X Install Data" directory. These are also used by the OS X Installer.
  9) Sets a variable in nvram that the OS X Installer uses to find the product install info after reboot.
  10) Uses the `bless` command to cause the Mac to boot from the kernel files copied to the "Mac OS X Install Data" directory.

Since most of the work is done with a postflight script, and since that script may need to do a lengthy copy of almost 4GB of data (if the package is not on the target volume), you may see a long delay at the "Running package scripts" stage of installation. This is normal.

The next step would be to reboot, but the postflight script does not do this; it just exits. The package is marked as requiring a reboot, so whatever mechanism is used to install the package is responsible for rebooting as soon as possible after the install.

Upon reboot, the machine boots and runs the OS X Installer just as if you had run the "Install Mac OS X Lion" application manually. It creates a "Recovery HD" partition and installs Lion on the target volume, displaying the OS X Installer GUI. When installation is complete, the machine reboots a second time, this time booting from the new Lion installation.


-----------------------
===Preinstall checks===
-----------------------
A distribution file at InstallLion.pkg/Contents/distribution.dist implements basic installation-check and volume-check logic, but it is nowhere as thorough as Apple's preinstall checks. It should prevent installation on most machines or volumes that don't meet the basic requirements for Lion.

The distribution.dist declares the install size is 8388608 KB (8* 1024 * 1024 KB, or 8GB), which should prevent attempted installation on volumes with less than 8GB free space.

*installation-check*
The installation-check script checks three hardware properties: the amount of RAM must be at least 2GB; the processor must be Intel, and the processor must have a 64bit-capable CPU.

*volume-check*
The volume check script checks if Mac OS X is already installed on the target volume. If it is, the script refuses to install on volumes with an OS X version less than 10.6.6 or greater than or equal to 10.8.


-----------------------------
===Customizing the install===
-----------------------------
There are several things you can do to customize the install.

*MacOSXInstaller.choiceChanges*
If a ChoiceChanges XML file named "MacOSXInstaller.choiceChanges" is included in the package at InstallLion.pkg/Contents/Resources/Mac OS X Install Data/, it will be copied to the "Mac OS X Install Data" directory on the target volume and used during the install.

*index.sproduct and additional packages*
When the "Install Mac OS X Lion" application runs, it queries Apple's Software Update Servers at http://swscan.apple.com/content/catalogs/others/index-lion-snowleopard-leopard.merged-1.sucatalog. In my testing, it then downloads a package named "MacOS_10_7_IncompatibleAppList.pkg" and copies it and an "index.sproduct" file that lists this package to the "Mac OS X Install Data" directory. This "MacOS_10_7_IncompatibleAppList.pkg" does not seem to be vital to the installation of Lion; in my testing, providing an "index.sproduct" file containing an empty "Packages" array was sufficient. The index.sproduct file must exist, however, or the automated install is aborted. I found also that I could not add arbitrary packages to the Packages array; the OS X Installer skipped any packages that were unsigned. If you find that you need one or more packages that are normally downloaded by the "Install Mac OS X Lion" application, you can include those packages together with a matching "index.sproduct" file in the InstallLion.pkg/Contents/Resources/Mac OS X Install Data/ directory, and the postflight script will copy them to the "Mac OS X Install Data" directory on the target volume where they will be used during the install.

*Additional packages*
The most likely customization you will want to do is to add additional packages to be installed after the OS install. Some examples might include a package that keeps the Setup Assistant from running when the machine first starts up under Lion, or a package that triggers your software installation management system to run, check for, and install any updates on the first boot after Lion is installed.

To get the OS X Installer to install additional packages, you must modify the InstallESD.dmg. You must add your additional packages and create an "OSInstall.collection" file that tells the OS X Installer which packages must be installed and the order in which to install them.

To make this customization easier, I've included a command-line tool: customizeInstallESD. This tool takes as input the original InstallESD.dmg and one or more additional packages. It outputs a new InstallESD.dmg file with your additional packages and and OSInstall.collection file added to the Packages directory.

-------------------------------
===Using customizeInstallESD===
-------------------------------
customizeInstallESD will look for the Install Mac OS X Lion.app in the Applications directory of the startup disk and all mounted volumes. If you've moved the application to a different location, or you have multiple versions (perhaps one for 10.7 and another for 10.7.1), you can tell customizeInstallESD to use a specific one by path using the -a or --app flags, or you can provide a path to an actual InstallESD.dmg file if you have one "outside" of the Install application.

The output disk image will also be named "InstallESD.dmg" and will be created in the current working directory by default. You can specify another directory and/or another disk image name with the --output flag.

Additional packages will be installed in the order they are provided to customizeInstallESD. 

Here's an example run of the tool, where I add four additional packages to be installed after Lion is installed:

% ./CustomizeInstallESD Packages/munkitools.mpkg Packages/munki_kickstart.pkg Packages/AdminAccount.pkg Packages/DisableSetupAssistant.pkg -o /Users/Shared/


Input DMG: /Volumes/Lion/Applications/Install Mac OS X Lion.app/Contents/SharedSupport/InstallESD.dmg

OSInstall.collection:
    (It's normal for OSInstall.mpkg to be listed twice)
------------------------------------------------------------------
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<array>
	<string>/System/Installation/Packages/OSInstall.mpkg</string>
	<string>/System/Installation/Packages/OSInstall.mpkg</string>
	<string>/System/Installation/Packages/munkitools.mpkg</string>
	<string>/System/Installation/Packages/munki_kickstart.pkg</string>
	<string>/System/Installation/Packages/AdminAccount.pkg</string>
	<string>/System/Installation/Packages/DisableSetupAssistant.pkg</string>
</array>
</plist>

------------------------------------------------------------------
Output DMG: /Users/Shared/InstallESD.dmg

Build custom installation DMG? [y/n] y

Note that you are given a chance to verify that all inputs and the output are as you expect before proceeding.

To install your customized Lion installation, simply copy the customized InstallESD.dmg instead of the "stock" disk image into the InstallLion.pkg. The customized image must still be named 'InstallESD.dmg' -- if you change the name, the install will fail. Keeping track of modified disk images all sharing the same name might get confusing, so take care. You can always mount a disk image and check the contents of the Packages directory if you lose track of your modified disk images.

===Notes on additional packages===

The Lion install environment is very stripped down. There are many command-line tools that are not available when booted into this environment. This can effect pre- and postflight scripts in packages. You may find that some packages that rely on pre- or postflight scripts to perform important tasks will fail to run properly in the Lion install environment. Check the install log at /var/log/install.log after the install is complete, or open the log window during installation to monitor pre- and postflight scripts.
This issue may limit the type and number of scripts you can use successfully in the Lion installation environment.

To get an idea what tools are available in the Lion install environment, boot into the Recovery HD. This environment is almost identical to the Lion install environment.
