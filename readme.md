### How to Setup Wayland for Android Emulation & Traffic Analysis w/ Mitmproxy

Below are the steps to setup an instance of Waydroid on a linux machine, with configuration settings for enabling local proxying on the same machine for traffic analysis using mitmproxy.

This assumes a completely fresh install of linux (tested on MX Linux w/ systemd), but should work for any flavors.

**Update repositories**
 
``` sudo apt update ```

**Install build tools**

``` sudo apt install build-essential ```

**Install Weston (Wayland Container)**

``` sudo apt install weston ```

**Install Waydroid (Android Emulator)**

``` sudo apt install curl ca-certificates -y ```

``` curl https://repo.waydro.id | sudo bash ```

``` sudo apt install waydroid -y ```

**Install ADB**

``` sudo apt install adb ```

**Download Android image checkpoint**

An eventual step is adding the mitmproxy certificate within the Android image.

Waydroid removed adb root from their system images in releases after Sept 2, 2023.

For now, the suggested method is to use the Sept 2, 2023 release.

The alternative is either enabling a shared folder between Android and the host, or creating the file manually within the cacerts folder. Steps for these are not built yet!

### Android Checkpoint Method

**Download Android system and vendor images**

This will download the system image from the Phoenix, AZ mirror

```curl --output system.zip https://phoenixnap.dl.sourceforge.net/project/waydroid/images/system/lineage/waydroid_x86_64/lineage-18.1-20230902-VANILLA-waydroid_x86_64-system.zip```

This will download the vendor image from the Las Vegas, NV mirror

``` curl --output vendor.zip https://versaweb.dl.sourceforge.net/project/waydroid/images/vendor/waydroid_x86_64/lineage-18.1-20230902-MAINLINE-waydroid_x86_64-vendor.zip ```

The above system / vendor images are for 64-bit operating systems (not Arm, not 32-bit, etc).

While this is typically the case for most desktop users wanting to hack away at apps, this will inevitably result in wanting to work with a particular Android app that is not supported (due to Android primarily running on ARM devices).

To resolve this, you can instead use a device with an ARM processor (such as a Raspberry Pi) with these steps, and use an ARM image instead of a 64-bit one.

You can find the separate images, as well as the different mirrors for each, here: https://sourceforge.net/projects/waydroid/files/images/

**Adding custom Android images**

The following commands require root access (on your device, not the emulator).

This assumes you opened a terminal and are in the directory of the system and vendor image:

``` sudo mkdir -p /etc/waydroid-extra/images```

``` sudo unzip system.zip -d /etc/waydroid-extra/images ```

``` sudo unzip vendor.zip -d /etc/waydroid-extra/images ```

**Launch Weston**

Weston will launch in a new window. In the top left of the window, there is a graphical button to click, which will open a terminal-like window.

To launch weston, type:

```weston```

**Weston will launch in a new window**
- In the top left of the window, there is a graphical button to click.
- Click this, it will open a terminal-like window.

Within the window, type the following command:

``` sudo waydroid init ```

This will go through a setup for waydroid, such as downloading the latest system image.

Once this is complete, use the following command (still in the terminal-like window):

``` waydroid session start ```

Select the same graphical button to open another terminal-like window, then, in the new window, run:

``` waydroid show-full-ui ```

You may see a few "failed attempt" or similar errors. However, this will keep waiting for the session to become live and will eventually start booting (less than a minute).

If all goes well, this should launch your android emulator and you will see a boot screen.

Depending on your hardware, the first boot (after seeing the boot screen) may take a minute or two.

**Enabling developer options in Android**
-- Access the Settings within Android (swipe or click & drag up to reveal the applications menu, then select Settings)
-- Scroll to the bottom of the settings and click the "About phone" menu item
-- Scroll down to the "Build number" and click this 5x
-- This will enable the developer mode and an alert will appear

**Enabling ADB root**
-- Go back to the main Settings screen
-- Click the System menu option
-- Click the "Developer options"
-- Under the "Debugging" section, toggle Rooted Debugging to Enabled

At this point, there should be a single terminal window and the weston window with two terminal-like windows running alongside the Android emulator.

Open a new terminal window or tab (outside Weston), then continue.

**Create iptable ruleset**

To enable internet access within the Waydroid instance, firewall settings need to be customized.

The below commands assume you are on wifi and are using one device to act as a client and server (in the case of request interception via mitmproxy) in the default configuration (port 8080).

If you are on ethernet, then the eth0 in these commands may be different. More details at the bottom of this.

``` sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 8080 ```

``` sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 443 -j REDIRECT --to-port 8080 ```

``` sudo ip6tables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 8080 ```

``` sudo ip6tables -t nat -A PREROUTING -i eth0 -p tcp --dport 443 -j REDIRECT --to-port 8080 ```

**Allow ports for Waydroid**

``` sudo ufw allow 67 ```

``` sudo ufw allow 53 ```

``` sudo ufw allow 8080 ```

``` sudo ufw default allow FORWARD ```


**Generate self signed certificate**

To intercept requests we need to take the mitmproxy and convert it into an acceptable format for Android.

To do this:

``` cd ~ && cd .mitmproxy ```

``` hashed_name=`openssl x509 -inform PEM -subject_hash_old -in mitmproxy-ca-cert.cer | head -1` && cp mitmproxy-ca-cert.cer $hashed_name.0 ```

**Connect Waydroid via ADB**

You will need to know the Android device IP.

This can be found by going to the main Settings menu on Android, then selecting "About phone".

There will be a section called "IP address" which will provide an address starting with **192.168.**

You will need to have previously set up the network information (ports / firewall / etc) for this to work, as it is being bridged from the host device.

Replacing **deviceIP** with the one you found:

``` sudo adb connect deviceIP:5555 ```

Proxies cannot be enabled within the Settings on Waydroid, and this is needed to pass requests over a specific port.

**Enabling the global proxy**

Note in the request below, the IP included is the one of your host device and the request should work as is.

``` adb shell settings put global http_proxy 192.168.240.1:8080 ```

**Root the Android Device**

``` adb root ```

**Make FS Rewritable**

```adb shell```

Root is required for the following steps, otherwise it won't work.

To confirm root, run the following in the shell:

``` whoami ```

You should see that you are running as root.

**Remount FS as writable**

``` mount -o remount,rw / ```

``` exit ```

The **exit** command will exit the shell and bring you back to terminal.

**Push self signed certificate**

ADB is connected to the device, but you will need to specify the cert path.

The cert being pushed is the one previously converted in the .mitmproxy folder.

This is easier when done within the ".mitmproxy" directory, which can be accessed using:

``` cd ~ && cd .mitmproxy ```

Assuming you are in the ".mitmproxy" directory:

``` adb push [certname] /system/etc/security/cacerts ```

Replace the [certname] above / below with the hashed name (ending in .0) within the .mitmproxy folder

**Set certificate permissions**

``` adb shell ```

``` cd /system/etc/security/cacerts ```

``` chmod 664 [certname] ```

``` exit ```

**Reboot device**

``` adb reboot ```

That is it!

You may lose root when you reboot, but that should not be necessary going forward.

If you need root again, simply run **adb root** as was done before.

### Help!
1) If you had problems getting root..

This can be due to the image downloaded from Wayland. The best thing to do is simply recheck your work, and, if all the steps were followed (enabling root within the developer settings, etc), then it is suggested to download an alternative checkpoint.

2) If you're using ethernet for connectivity instead of wifi..

Then the **eth0** in the firewall settings may not be accurate.

It is suggested to run the command **ifconfig** to see the current network configurations.
Waydroid may be prefaced with **eth0**, **eth1**, or another variant if there are multiple configurations.
