#Firstly install ALSA and Loopback Device modules
sudo apt update
sudo apt install linux-modules-extra-$(uname -a)
sudo echo "snd-aloop" >> /etc/modules
sudo modprobe snd-aloop
sudo lsmod | grep snd_aloop
If the output shows the snd-aloop module loaded, then the ALSA loopback configuration step is complete.

Install ffmpeg
sudo apt-get update
sudo apt-get install ffmpeg
Install stable chrome and chrome-driver
curl -sS -o - https://dl-ssl.google.com/linux/linux_signing_key.pub | apt-key add
echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" > /etc/apt/sources.list.d/google-chrome.list
sudo apt-get -y update
sudo apt-get -y install google-chrome-stable
sudo mkdir -p /etc/opt/chrome/policies/managed
sudo echo '{ "CommandLineFlagSecurityWarningsEnabled": false }' >>/etc/opt/chrome/policies/managed/managed_policies.json


CHROME_DRIVER_VERSION=`curl -sS chromedriver.storage.googleapis.com/LATEST_RELEASE`
wget -N http://chromedriver.storage.googleapis.com/$CHROME_DRIVER_VERSION/chromedriver_linux64.zip -P ~/
unzip ~/chromedriver_linux64.zip -d ~/
rm ~/chromedriver_linux64.zip
sudo mv -f ~/chromedriver /usr/local/bin/chromedriver
sudo chown root:root /usr/local/bin/chromedriver
sudo chmod 0755 /usr/local/bin/chromedriver
Install Miscelleneous things:
sudo apt-get install default-jre-headless ffmpeg curl alsa-utils icewm xdotool xserver-xorg-input-void xserver-xorg-video-dummy
Install Jibri
wget -qO - https://download.jitsi.org/jitsi-key.gpg.key | sudo apt-key add -
sudo sh -c "echo 'deb https://download.jitsi.org stable/' > /etc/apt/sources.list.d/jitsi-stable.list"
sudo apt-get update
sudo apt-get install jibri

sudo usermod -aG adm,audio,video,plugdev jibri
Confiuration of Jitsi
Config Prosody:
Please add the following componant in /etc/prosody/prosody.cfg.lua or /etc/prosody/conf.d/yourdomain.com.cfg.lua

Component "internal.auth.yourdomain.com" "muc"
    modules_enabled = {
      "ping";
    }
    storage = "null"
    muc_room_cache_size = 1000
    
VirtualHost "recorder.yourdomain.com"
  modules_enabled = {
    "ping";
  }
  authentication = "internal_plain"
Restart prosody
sudo service prosody restart
Register two users through prosodyctl
prosodyctl register jibri auth.yourdomain.com
prosodyctl register recorder recorder.yourdomain.com
let me assume password of first and second user are respectively jibriauthpass and jibrirecorderpass

Configure Jicofo
Edit /etc/jitsi/jicofo/sip-communicator.properties and add this lines

org.jitsi.jicofo.jibri.BREWERY=JibriBrewery@internal.auth.yourdomain.com
org.jitsi.jicofo.jibri.PENDING_TIMEOUT=90
Restart Jicofo
sudo service jicofo restart
Configure Jitsi Meet
Edit the /etc/jitsi/meet/yourdomain.config.js file, add/set the following properties:

fileRecordingsEnabled: true, // If you want to enable file recording
liveStreamingEnabled: true, // If you want to enable live streaming
hiddenDomain: 'recorder.yourdomain.com',
Configuration of Jibri
Download finalize_recording.sh

wget https://raw.githubusercontent.com/jitsi/jibri/master/resources/finalize_recording.sh
if this is not available this is the code, you can create file called finalize_recording.sh and put the following code in that and save the file

#!/bin/bash

RECORDINGS_DIR=$1

echo "This is a dummy finalize script" > /tmp/finalize.out
echo "The script was invoked with recordings directory $RECORDINGS_DIR." >> /tmp/finalize.out
echo "You should put any finalize logic (renaming, uploading to a service" >> /tmp/finalize.out
echo "or storage provider, etc.) in this script" >> /tmp/finalize.out

exit 0

Configure Jibri

Edit Jibri config in /etc/jitsi/jibri/config.json

{
    // NOTE: this is a *SAMPLE* config file, it will need to be configured with
    // values from your environment

    // Where recording files should be temporarily stored
    "recording_directory":"/tmp/recordings",
    // The path to the script which will be run on completed recordings
    "finalize_recording_script_path": "/home/user/finalize_recording.sh",
    "xmpp_environments": [
        {
            // A friendly name for this environment which can be used
            //  for logging, stats, etc.
            "name": "prod environment",
            // The hosts of the XMPP servers to connect to as part of
            //  this environment
            "xmpp_server_hosts": [
                "localhost"
            ],
            // The xmpp domain we'll connect to on the XMPP server
            "xmpp_domain": "yourdomain.com",
            // Jibri will login to the xmpp server as a privileged user 
            "control_login": {
                // The domain to use for logging in
                "domain": "auth.yourdomain.com",
                // The credentials for logging in
                "username": "jibri",
                "password": "jibriauthpass"
            },
            // Using the control_login information above, Jibri will join 
            //  a control muc as a means of announcing its availability 
            //  to provide services for a given environment
            "control_muc": {
                "domain": "internal.auth.yourdomain.com",
                "room_name": "JibriBrewery",
                "nickname": "jibri-nickname"
            },
            // All participants in a call join a muc so they can exchange
            //  information.  Jibri can be instructed to join a special muc
            //  with credentials to give it special abilities (e.g. not being
            //  displayed to other users like a normal participant)
            "call_login": {
                "domain": "recorder.yourdomain.com",
                "username": "recorder",
                "password": "jibrirecorderpass"
            },
            // When jibri gets a request to start a service for a room, the room
            //  jid will look like:
            //  roomName@optional.prefixes.subdomain.xmpp_domain
            // We'll build the url for the call by transforming that into:
            //  https://xmpp_domain/subdomain/roomName
            // So if there are any prefixes in the jid (like jitsi meet, which
            //  has its participants join a muc at conference.xmpp_domain) then
            //  list that prefix here so it can be stripped out to generate
            //  the call url correctly
            "room_jid_domain_string_to_strip_from_start": "conference.",
            // The amount of time, in minutes, a service is allowed to continue.
            //  Once a service has been running for this long, it will be
            //  stopped (cleanly).  A value of 0 means an indefinite amount
            //  of time is allowed
            "usage_timeout": "0"
        }
    ]
}
Restart Jibri
sudo systemctl restart jibri
