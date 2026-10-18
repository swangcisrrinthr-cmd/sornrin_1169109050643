# Firmware and flow checks

The firmware uses one Arduino `loop()`: check the WiFi button, try MQTT if
needed, process incoming commands, then read/update/publish sensor data when
due. There are no application-created tasks, queues or atomic flags.
`reconnect()` attempts a connection once and uses `millis()` to wait five
seconds after that attempt before trying again. It has no retry `while` or
five-second `delay`, so the loop keeps checking the button between attempts.

TCP connect and MQTT response timeouts are each set to one second in the
installed ESP32 core and PubSubClient. Connection calls are still synchronous:
DNS can take longer, and a short button press during a network wait can be
missed. An IP address for `mqtt_server` avoids DNS lookup. Sensor timing is
approximate and can be delayed by these waits. Offline samples are skipped.
The blocking WiFi portal disconnects MQTT and pauses sensor updates until it
closes; it retains its 180-second timeout and restart behavior.

Relay commands are validated directly in `mqttCallback()` in `src/main.cpp`,
before any GPIO write or ACK. They must contain `cmd` equal to `pump` or
`valve` and integer `state` equal to `0` or `1`. Invalid requests do not change GPIOs or produce
a success ACK. Valid ACKs report the GPIO level after the write. The Node-RED
dashboard updates each relay switch from its ACK and periodic telemetry;
feedback messages update the UI only and are not republished as commands.

## Automated checks

- `platformio run` builds the ESP32 firmware.
- `node --test test/flows.test.cjs` checks flow IDs, references and routing.

Command validation is now part of the Arduino callback and has no standalone
native parser test. Verify its behavior using the hardware checks below.

## Hardware checks after uploading

1. Start with the broker unavailable. Press and hold GPIO 4's button:
   `WiFi Config Mode` and the `parinya_AP` portal should appear after any
   in-progress connection wait ends. MQTT retries stop while the portal is
   open. Repeat with DNS unavailable and record the longer response delay;
   do not expect a short press during a network wait to be detected.
2. Release the button. Configure WiFi and check that the portal closes,
   MQTT reconnects, subscriptions resume, and telemetry reaches Node-RED.
3. Confirm holding the button through portal completion does not reopen it
   until it has been released and pressed again.
4. With test loads disconnected, verify valid pump/valve commands and ACKs;
   empty payloads, payloads over 256 bytes, malformed JSON, invalid command
   names, missing fields, string/boolean/fractional states and values outside
   0/1 must not change outputs or return a success ACK.
5. Leave the broker offline without opening the portal. Check that sensor/LCD
   updates continue between attempts, and connection logs show a five-second
   gap after each failed attempt. Restore the broker and verify commands,
   ACKs and telemetry resume. Repeat using a broker IP to exclude DNS waits.

Automated tests do not substitute for these board and wiring checks.
