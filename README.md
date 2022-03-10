# StreamDeck-AudioSwitcher-workaround

## Summary

This is a ridiculously involved workaround for a little bug in https://github.com/fredemmott/StreamDeck-AudioSwitcher that creates duplicate usb device entries after reboot.

Issue: https://github.com/fredemmott/StreamDeck-AudioSwitcher/issues/56

## Instructions

WARNING: Please back up your configurations before running any of this. Also note that when I was debugging the launch agent code, it would often freeze the OS and I would have to force restart often. However, now that it's running, it's quite stable.

Create script that will replace the device ID. Place it in /usr/local/bin.

`/usr/local/bin/fix-audio-mapping.sh`
```bash
#!/bin/bash
# For reference: https://dev.to/anesabml/autostart-scrcpy-on-device-connection-39a

# Prerequisites:
# brew install jq
# brew install switchaudio-osx

# Usage:
# Update vars
#   STREAMDECK_PROFILE - path to your streamdeck profile manifest
#   DEVICE_JSON_PATH - json path to the 
#   DEVICE_GREP_STR - a grep search for your usb device
#      Will be searched like this:
#      $ SwitchAudioSource -f json -a | grep -i SEARCHTERM

STREAMDECK_PROFILE="Users/justin/Library/Application Support/com.elgato.StreamDeck/ProfilesV2/A380B6A4-A474-4433-8AA5-C1AF5BFB5054.sdProfile/manifest.json"
DEVICE_JSON_PATH='.Actions."1,2".Settings.secondary'
DEVICE_GREP_STR='vanatoo'

jq() {
  /opt/homebrew/bin/jq "$@"
}
export -f jq

SwitchAudioSource() {
  /opt/homebrew/bin/SwitchAudioSource "$@"
}
export -f SwitchAudioSource

# Redirect std out to a log file
exec 1> ~/output.log 2>&1

date # To timestamp the run in the log

CURRENT_DEVICE=$(
cat "$STREAMDECK_PROFILE" | jq $DEVICE_JSON_PATH
)

DETECTED_DEVICE=$(
sleep 1
SwitchAudioSource -f json -a | grep -i $DEVICE_GREP_STR | jq '.uid="output/\(.uid)" | .uid'
)

echo "Detected"
echo $DETECTED_DEVICE

if [ -z "$DETECTED_DEVICE" ];
then
  echo "Search device not found. Skipping..."
else
  echo "Search device found!"

  if [ "$DETECTED_DEVICE" == "$CURRENT_DEVICE" ];
  then
    echo "No change needed"
  else
    echo "Change needed!"

    echo "Updating from"
    echo $CURRENT_DEVICE
    echo "to"
    echo $DETECTED_DEVICE

    killall Stream\ Deck

    # Update the value
    jq '. | .Actions."1,2".Settings.secondar = "testing"'
    tmpfile=$(mktemp)
    cat "$STREAMDECK_PROFILE" | jq ". | $DEVICE_JSON_PATH = $DETECTED_DEVICE" > ${tmpfile}
    cp "$STREAMDECK_PROFILE"  "$STREAMDECK_PROFILE.bak"
    cat ${tmpfile} > "$STREAMDECK_PROFILE"

    # The -g to not bring to foreground isn't respected for some reason
    open -g /Applications/Stream\ Deck.app
  fi
fi


```

Create a Launch Agent:
This is run a command everytime a specific USB device is connected. I set this to run every time my speaker connection is detected.

`~Library/LaunchAgents/com.streamdeck.audiofix.plist`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.streamdeck.audiofix</string>
  <key>ProgramArguments</key>
  <array>
    <!--
      For this first argument, see
      https://dev.to/anesabml/autostart-scrcpy-on-device-connection-39a
      https://github.com/himbeles/mac-device-connect-daemon

    Summary:
          git clone https://github.com/himbeles/mac-device-connect-daemon
          gcc -framework Foundation -o xpc_set_event_stream_handler xpc_set_event_stream_handler.m
          cp xpc_set_event_stream_handler /usr/local/bin/
    -->
    <string>/usr/local/bin/xpc_set_event_stream_handler</string> 
    <string>/usr/local/bin/fix-audio-mapping.sh</string>
  </array>
  <key>LaunchEvents</key>
  <dict>
    <key>com.apple.iokit.matching</key>
    <dict>
      <key>com.apple.device-attach</key>
      <dict>
        <key>idProduct</key>
        <integer>18</integer> <!-- Determine with `ioreg -p IOUSB -l` -->
        <key>idVendor</key>
        <integer>3468</integer> <!-- Determine with `ioreg -p IOUSB -l` -->
        <key>IOProviderClass</key>
        <string>IOUSBDevice</string>
        <key>IOMatchLaunchStream</key>
        <true/>
      </dict>
    </dict>
  </dict>
</dict>
</plist>
```

Load the launch agent:

```bash
launchctl load ~/Library/LaunchAgents/com.streamdeck.audiofix.plist
```

The script will now run and replace the usb id with the updated id.

