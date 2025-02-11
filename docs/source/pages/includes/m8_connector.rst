Auxiliary M8 connector
----------------------

M8 aux connector has 8pin female A-coded connector. M12 ethernet has 8pin female X-coded connector.

.. image:: /_static/images/poe/m8-connector.jpeg

.. list-table::
   :header-rows: 1

   * - Pin
     - Function
     - Additional information
   * - 1
     - AUX GPIO 3V3
     - 3V3 auxiliary GPIO **52**. Pull this pin high with 10k resistor at startup for **USB boot**.
   * - 2
     - FSYNC ISO
     - Isolated Frame synchronization input signal for cameras
   * - 3
     - USB D+
     - USB 2.0 interface used as a host to control external devices. Could be used to connect to the device (eg. reflash), if USB boot is enabled (via AUX GPIO 3V3).
   * - 4
     - USB D-
     - USB 2.0 interface used as a host to control external devices. Could be used to connect to the device (eg. reflash), if USB boot is enabled (via AUX GPIO 3V3).
   * - 5
     - VBUSIN
     - This pin is used for sourcing 5V power to accessory devices connected to the M8 connector. If trying to boot the device in USB boot mode, this pin can also sink current.
   * - 6
     - Strobe ISO
     - Isolated Strobe output signal for cameras (Open Collector)
   * - 7
     - Isolated GND
     - Isolated ground for Strobe/FSYNC
   * - 8
     - GND
     - Ground.

Cameras also include an M8 connector cap for waterproofing in case the M8 connector wouldn't be used.

.. image:: /_static/images/guides/m8-isolated-circiut.png

**External triggering**: Sensors require 1V8 rising edge on FSYNC for the trigger event. We are using optocoupler and ESD protection diode, so the input trigger voltage can be up to 24V
and trigger logic is reversed, so trigger event happens at 0V on FSYNC line.