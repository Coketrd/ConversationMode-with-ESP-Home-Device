# NoWakeWord_DoAConversation
Do a Conversation with youre HomeAssistant-Assist without needing a WakeWord the hole time.

This is not a finished integration. The result is achieved using various scripts and helpers. I give no guarantee that this variant will work for you. Everyone is responsible for checking the code and any damage it may cause.

First of all, I would like to tell you about my setup. It may be that this also works with other systems, but I have the following setup:

- An ESP32 with an INMP441 microphone
- HomeAssitant software
- Extended OpenAI Conversation Integration
- Alexa Media Player Integration
- The Home Assistant Cloud
- Amazon Echo Dot for playback

What is the procedure?

The ESP32 records the audio, which is converted into text and sent to chatGPT for answering. This then sends back its answer in the form of text. The text is then spoken in an mp3 file with the selected voice from the cloud and stored locally on the system. This link is tapped, saved in a helper and the media file is temporarily copied via a pyscript, opened and the length of the mp3 file is read out. Depending on how long the MP3 file is, additional seconds are added and stored in a timer.

As soon as the link changes (which happens with every new request), the pyscript is started via an automation and waits for the timer to expire. As soon as the timer has expired, listening is reactivated by the ESP.

What needs to be created?

1. For this purpose, an additional switch for immediate listening must be integrated into the ESP, which simultaneously deactivates the wakeword during this time. 
This is the code for the ESP. I use an ESP32 NodeMCU Module WLAN WiFi Dev Kit C Development Board with CP2102

```
esphome:
  name: "esp32-microphone-assist"
  friendly_name: "esp32-microphone-assist"

esp32:
  board: esp32dev
  framework:
    type: arduino

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "YOURE API-CODE"

ota:
  - platform: esphome
    password: "YOURE OTA-CODE"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Test Fallback Hotspot"
    password: "Youre-WIFI-PASSWORD"

captive_portal:

i2s_audio:
  - id: i2s_in
    i2s_lrclk_pin: GPIO26 #WS 
    i2s_bclk_pin: GPIO25 #SCK

microphone:
  - platform: i2s_audio
    adc_type: external
    pdm: false
    id: mic_i2s
    channel: right
    bits_per_sample: 32bit
    i2s_audio_id: i2s_in
    i2s_din_pin: GPIO33  #SD Pin from the INMP441 Microphone


voice_assistant:
  microphone: mic_i2s
  id: va
  noise_suppression_level: 2
  auto_gain: 31dBFS
  volume_multiplier: 4.0
  use_wake_word: false
  

# I don´t have connected the lights
  
#  on_wake_word_detected: 
#    - light.turn_on:
#        id: led_light
#  on_listening: 
#    - light.turn_on:
#        id: led_light
#        effect: "Scan Effect With Custom Values"
#        red: 63%
#        green: 13%
#        blue: 93%
  
#  on_stt_end:
#    - light.turn_on:
#        id: led_light
#        effect: "None"
#        red: 0%
#        green: 100%
#        blue: 0%

  on_error: 
#    - light.turn_on:
#        id: led_light
#        effect: "None"
    - if:
        condition:
          switch.is_on: use_wake_word
        then:

          - switch.turn_off: use_wake_word
          - delay: 1sec 
          - switch.turn_on: use_wake_word

  on_tts_start: 
    - homeassistant.service: 
        service: script.ha_voice_alexa
        data: 
          media_player_id: 'media_player.echo_buro'    # Here you must enter your media-player ID, which should be linked to the ESP microphone.
          tts_id: 'tts.home_assistant_cloud'           # You must enter your TTS ID here. For me it is the cloud
          lang: 'de-DE'                                # The Language you want
          voice_agent: 'ChristophNeural'               # The voice you want
          messagecall: !lambda 'return x;'
          
# Here we get the URL with which the mp3 file is saved and send it to the script.
  on_tts_end:
    - homeassistant.service: 
        service: script.ha_voice_alexa_url
        data:
          url: !lambda 'return x;'

        

  on_client_connected:
    - if:
        condition:
          switch.is_on: use_wake_word
        then:
          - voice_assistant.start_continuous:

  on_client_disconnected:
    - if:
        condition:
          switch.is_on: use_wake_word
        then:
          - voice_assistant.stop:

# I didnt use lights
 
#  on_end:
#    - light.turn_off:
#        id: led_light


binary_sensor:
  - platform: status
    name: API Connection
    id: api_connection
    filters:
      - delayed_on: 1s
    on_press:
      - if:
          condition:
            switch.is_on: use_wake_word
          then:
            - voice_assistant.start_continuous:
    on_release:
      - if:
          condition:
            switch.is_on: use_wake_word
          then:
            - voice_assistant.stop:

# This is the wakeword-Switch
switch:
  - platform: template
    name: Use wake word
    id: use_wake_word
    optimistic: true
    restore_mode: RESTORE_DEFAULT_ON
    entity_category: config
    on_turn_on:
      - lambda: id(va).set_use_wake_word(true);
      - if:
          condition:
            not:
              - voice_assistant.is_running
          then:
            - voice_assistant.start_continuous
    
    on_turn_off:
      - voice_assistant.stop
      - lambda: id(va).set_use_wake_word(false);

# This is the listen now Switch which we use to take a conversation with the assist.     
  - platform: template
    id: listen_once
    name: Listen Once
    optimistic: true
    restore_mode: ALWAYS_OFF
    on_turn_on:
      - switch.turn_off: use_wake_word
      - delay: 0.5s
      - voice_assistant.start
    on_turn_off:
      - switch.turn_on: use_wake_word  

#light:
#  - platform: neopixelbus
#    id: led_light
#    type: grb
#    pin: GPIO32      # DIN pin of the LED Strip
#    num_leds: 9      # change the Number of LEDS according to your LED Strip.
#    name: "Light"
#    variant: ws2812x
#    default_transition_length: 0.5s
      
#    effects:
#      - addressable_scan:
#          name: Scan Effect With Custom Values
#          move_interval: 50ms
#          scan_width: 2

```

2. We need to create helper:
   - in which we save the URL.
     i Useing an Input Text Helper:   input_text.url_esp32_voice_assist
   - in which we save the timer
     i Using an timer Helper:   timer.extract_mp3_time
   - with which we activate continuous listening. Only when this helper is active are the mp3 files analyzed and the microphone reactivated at the end of the voice output. You can activate this helper directly via the voice assistant by saying: activate continuous listening when you first enter your voice. Or you can link the assistant to a switch on a wall or simply
     activate and deactivate it via smartphone/tablet/watch.
     This is my helper: input_boolean.voice_assistant_permanent_listening_active

4. We need to create a script to save the URL in the helper

```
alias: ha_voice_alexa_url
sequence:
  - target:
      entity_id: input_text.url_esp32_voice_assist
    data:
      value: "{{ url }}"
    action: input_text.set_value
mode: single
description: ""
```
5. Now you need to install the integration pyscript from HACS https://github.com/custom-components/pyscript and set:
   - Allow all imports? YES.
   - Use Home Assistant as a global variable? YES

6. Make a new pyscript with the code:
```
@service
async def download_and_get_duration(mp3_url=None):
    import aiohttp
    import io
    import os
    from mutagen.mp3 import MP3

    output_path = "/config/pyscript/test_download2.mp3"

    async def debug_print(message):
        print(message)  # Use print for protocol output

    if not mp3_url:
        debug_print("No MP3 URL provided.")
        return

    try:
        # Download the MP3 file
        async with aiohttp.ClientSession() as session:
            async with session.get(mp3_url) as response:
                if response.status == 200:
                    debug_print("File downloaded successfully.")
                    content = response.read()  # Read the data from the answer

                    # Write the file to the hard disk
                    with io.open(output_path, 'wb') as f:
                        f.write(content)

                    # Check whether the file has been saved
                    if os.path.exists(output_path):
                        debug_print(f"Temporary file was successfully saved under {output_path}.")
                    else:
                        debug_print(f"Temporary file was not found under {output_path}.")
                        return

                    # Read out the duration of the MP3 file
                    try:
                        audio = MP3(output_path)
                        duration_seconds = audio.info.length
                        debug_print(f"The duration of the MP3 file is {duration_seconds} seconds.")

                        # Add additional seconds based on the length
                        if duration_seconds < 10:
                            duration_seconds += 4
                        elif 10 <= duration_seconds < 20:
                            duration_seconds += 6
                        elif 20 <= duration_seconds < 30:
                            duration_seconds += 8

                        debug_print(f"The adjusted duration is {duration_seconds} seconds.")

                        # Setting the duration in the timer
                        service.call("timer", "start", entity_id="timer.extract_mp3_time", duration=duration_seconds)
                    except Exception as e:
                        debug_print(f"Error when reading out the MP3 duration: {e}")
                else:
                    debug_print(f"Error downloading the file. Status code: {response.status}")
    except Exception as e:
        debug_print(f"Fehler im Skript: {e}")
```

7. Now we need an automation to activate the automatic listening
   
```
alias: Activate continuous listening.
description: ""
trigger:
  - platform: state
    entity_id:
      - input_text.url_esp32_voice_assist
    for:
      hours: 0
      minutes: 0
      seconds: 0
condition:
  - condition: state
    entity_id: input_boolean.voice_assistant_permanent_listening_active
    state: "on"
    enabled: true
action:
  - data:
      mp3_url: "{{ states('input_text.url_esp32_voice_assist') }}"
    action: pyscript.download_and_get_duration
  - wait_for_trigger:
      - platform: state
        entity_id:
          - timer.extract_mp3_time
        to: idle
  - type: turn_on
    device_id: #YOURE DEVICE ID. FOR EXAMPLE esp32-microphone-assist
    entity_id: #YOure SWITCH for listening on ESP. FOR EXAMPLE: Listen Once
    domain: switch
    enabled: true
mode: single
```
8. We need a second automation to reset the listen once switch from the esp. This ensures that the wakeword is active again for further entries until the end of the next voice output.

```
alias: Listen-Once-ESP32-Assist deaktivieren
description: ""
trigger:
  - platform: device
    type: turned_on
    device_id: #YOURE DEVICE ID. FOR EXAMPLE esp32-microphone-assist
    entity_id: #YOure SWITCH for listening on ESP. FOR EXAMPLE: Listen Once
    domain: switch
    for:
      hours: 0
      minutes: 0
      seconds: 3
condition: []
action:
  - type: turn_off
    device_id: #YOURE DEVICE ID. FOR EXAMPLE esp32-microphone-assist
    entity_id: #YOure SWITCH for listening on ESP. FOR EXAMPLE: Listen Once
    domain: switch
mode: single
```
9. So that you can see when listening is active again and you know when you can speak again, we need some kind of indicator, which could be a light, a beep or something else. For example, I used the light from a socket for this.

If anyone knows how to control the light ring of the EchoDot and use it for this purpose, I would be grateful for the code. I haven't found anything about it.
   

