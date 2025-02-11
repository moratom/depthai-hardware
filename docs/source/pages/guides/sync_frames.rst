.. _sync_frames:

Hardware message syncing
########################

There are 2 way to synchronize messages from different sensors (frames, IMU packet, ToF, etc.);

- :ref:`Hardware syncing <Synchronizing frames externally>` (multi-sensor sub-ms accuracy, hardware trigger)
- `Software syncing <https://docs.luxonis.com/projects/api/en/latest/tutorials/message_syncing/>`__ (based on timestamp/sequence numbers)

This documentation page focuses on **hardware syncing**, which allows precise synchronization across multiple
camera sensors and potentially with other hardware, e.g. flash LED, external IMU, or other cameras.

FSYNC signal
------------

**FSYNC/FSIN** (frame sync) signal is a pulse that is driven high at the start of each frame capture. Its length is not proportional
to the exposure time. It can be either **input or output**. It operates in **1.8V** logic.

On stereo cameras (OAK-D*) we want stereo camera pair (monochrome cameras) to be perfectly in sync, so one
camera sensor (eg. left) has FSYNC set to INPUT, while the other camera sensor (eg. right)
has FSYNC set to OUTPUT. In such configuration the right camera drives left camera.

.. note::

    At the moment, only OV9282/OV9782 can output FSYNC signal, while IMX378/477/577/etc should also have the
    capability, but isn't yet supported (so these can not drive FSYNC signal, only be driven by it).
    AR0234 has input-only FSYNC trigger.

Synchronizing frames externally
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If we would like to drive cameras with an outside signal, we would need to set FSIN as INPUT for camera sensors.

All `Series 2 OAK PoE models <https://docs.luxonis.com/projects/hardware/en/latest/pages/articles/oak-s2.html>`__ have an M8 I/O connector
which exposes FSIN signal (and also STROBE). So you could connect a signal generator to an M8 connector and all 3
camera sensors would capture a frame based on the signal generator triggers.

.. code-block:: python

    # Example: we have 3 cameras on ports A,B, and C
    cam_A.initialControl.setFrameSyncMode(dai.CameraControl.FrameSyncMode.INPUT)
    cam_B.initialControl.setFrameSyncMode(dai.CameraControl.FrameSyncMode.INPUT)
    cam_C.initialControl.setFrameSyncMode(dai.CameraControl.FrameSyncMode.INPUT)

You can also control FSIN line via GPIO from within `Script node <https://docs.luxonis.com/projects/api/en/latest/components/nodes/script/>`__,
see example `here <https://gist.github.com/Erol444/a9189a8215371ff9f4cf4472960e1d66>`__.

STROBE signal
-------------

**STROBE signal** is an **output** from the image sensor, and is active (high) during the exposure of the image sensor.
It would be used to eg. drive an external LED lighting for illumination - so lighting would only be active during exposure
times, instead constantly on, which would decrease power consumption and heating of the lighting.

We have used STROBE signal on :ref:`Pro version` of OAK cameras (which have on-board illumination IR LED and IR laser dot
projector) to drive the laser/LED.

Frame capture graphs
--------------------

Frame timestamp is assigned to the frame at the MIPI SoF (start of frame) event, when the sensor starts streaming the frame
(MIPI readout).

For global shutter sensors, this follows immediately after the exposure for the whole frame was finished,
so we can say the timestamp assigned is aligned with end of exposure-window (within a margin of few microseconds).
Here's an example graph of the global shutter sensor timings, which demonstrates when timestamp is assigned to the frame:

.. code-block::

    frame-time |---------33.3ms---------|---------33.3ms---------|---------33.3ms---------|-

    exposure           <-----20ms------>                 <-10ms->     <-------30ms------->
              _        _________________                 ________     ____________________
    STROBE     |______|     frame1      |_______________| frame2 |___|      frame3        |_
                _                        _                        _                        _
    FSIN      _| |______________________| |______________________| |______________________|

    MIPI      _     frame0  ____________    frame1   ____________   frame2    ____________
    readout    \XXXXXXXXXXX/            \XXXXXXXXXXX/            \XXXXXXXXXXX/            \

    capture    ^                        ^                        ^                        ^
    timestamp  frame0                   frame1                   frame2                   frame3

For rolling shutter, the example graph looks a bit different.
MIPI SoF follows after the first row of the image was fully exposed and it's being streamed, but the following rows are
still exposing or may have not started exposing yet (depending on exposure time).

Below there's an example graph of rolling shutter sensor (IMX378) at 1080p
and 30fps (33.3ms frame time). MIPI readout time varies between sensors/resolutions, but for IMX378 it's 16.54ms at 1080P,
23.58ms at 4K, and 33.04ms at 12MP.

.. code-block::

    frame-time |---------33.3ms---------|---------33.3ms---------|---------33.3ms---------|-------------...

    exposure           frame1                            frame2       frame3
    exposure  row0     <-----20ms------>                 <-10ms->     <-------30ms------->
    exposure  row540         <-----20ms------>                 <-10ms->     <-------30ms------->
    exposure  row1079              <-----20ms------>                 <-10ms->     <-------30ms------->

    MIPI      _     frame0  ____________    frame1   ____________   frame2    ____________   frame3    _...
    readout    \XXXXXXXXXXX/            \XXXXXXXXXXX/            \XXXXXXXXXXX/            \XXXXXXXXXXX/
                16.54ms                  16.54ms                  16.54ms                  16.54ms
    capture    ^                        ^                        ^                        ^
    timestamp  frame0                   frame1                   frame2                   frame3
                _                        _                        _                        _
    FSYNC     _| |______________________| |______________________| |______________________| |___________...


OAK-FFC hardware syncing
========================

On :ref:`OAK-FFC-4P`, we have 4 camera ports; A (rgb), B (left), C (right), and D (cam_d). A & D are 4-lane MIPI, and B & C
are 2-lane MIPI. Each pair (A&D and B&C) share an I2C bus, and the B&C bus is configured for HW syncing left+right cameras by default.

For A&D ports, you need to explicitly enable hardware syncing:

.. code-block:: python

    cam_A.initialControl.setFrameSyncMode(dai.CameraControl.FrameSyncMode.OUTPUT)
    cam_D.initialControl.setFrameSyncMode(dai.CameraControl.FrameSyncMode.INPUT)

Arducam FFC camera syncing
--------------------------

Arducam FFC cameras have a 22-pin connector, which **don't have lines for FSIN/STROBE**. As seen below, to connect
Arducam FFC camera to our OAK-FFC baseboard you need to use 26-to-22 pin converter connector which only exposes
STROBE/FSIN lines via test pads. To sync these cameras, you could either solder a wire from test pad to
the camera module's FSIN header pin, or connect all FSIN header pins together, as done
`here <https://discuss.luxonis.com/d/934-ffc-4p-hardware-synchronization/3>`__.

.. image:: https://user-images.githubusercontent.com/18037362/196110419-27639d8f-ea88-4945-8fa0-99fa91061f07.jpg

Connecting FSIN/STROBE
----------------------

As mentioned, all `Series 2 OAK PoE models <https://docs.luxonis.com/projects/hardware/en/latest/pages/articles/oak-s2.html>`__ have an M8 I/O connector
with FSYNC/STROBE signal. But if you won't be using these, you will likely need to solder a wire to the PCB on your device. Most PCB designs
are open-source (on `depthai-hardware <https://github.com/luxonis/depthai-hardware>`__ repository), so you can easily check where FSIN/STROBE
signals are on the PCB.

OAK-FFC-4P FSIN
^^^^^^^^^^^^^^^

.. image:: https://user-images.githubusercontent.com/18037362/202100246-53ed754f-e2b3-450f-8deb-538589ab07fb.png

As shown on image above, on :ref:`OAK-FFC-4P` you can enable connection of ``FSIN_4LANE`` and ``FSIN_2LANE`` with the MXIO6.
The script below will sync together all 4 cameras that are connected to the OAK-FFC-4P.

.. code-block:: python

    # CAM_A will drive FSIN signal for all other cameras:
    cam_A.initialControl.setFrameSyncMode(dai.CameraControl.FrameSyncMode.INPUT) # 4LANE
    cam_B.initialControl.setFrameSyncMode(dai.CameraControl.FrameSyncMode.OUTPUT) # 2LANE
    cam_C.initialControl.setFrameSyncMode(dai.CameraControl.FrameSyncMode.INPUT) # 2LANE
    cam_D.initialControl.setFrameSyncMode(dai.CameraControl.FrameSyncMode.INPUT) # 4LANE

    # AND importantly to tie the FSIN signals of A+D and B+C pairs, by setting a GPIO:
    # OAK-FFC-4P requires driving MXIO6 high (FSIN_MODE_SELECT) to connect together
    # the A+D FSIN group (4-lane pair) with the B+C group (2-lane pair)
    config = dai.Device.Config()
    config.board.gpio[6] = dai.BoardConfig.GPIO(dai.BoardConfig.GPIO.OUTPUT,
                                                dai.BoardConfig.GPIO.Level.HIGH)

    with dai.Device(config) as device:
        device.startPipeline(pipeline)

Additional info can be found in `this forum discussion <https://discuss.luxonis.com/d/934-ffc-4p-hardware-synchronization/3>`__.

Series 2 USB OAKs
^^^^^^^^^^^^^^^^^

FSIN lines on DM9098 board (:ref:`OAK-D S2`, :ref:`OAK-D W`, :ref:`OAK-D Pro`, :ref:`OAK-D Pro W`):

.. image:: https://user-images.githubusercontent.com/18037362/202097546-af3f3035-b4a3-4be2-8136-d37c2ee9d622.png

USB OAK-1* FSIN
^^^^^^^^^^^^^^^

FSIN test pad on NG9093 board (:ref:`OAK-1`, :ref:`OAK-1 W`, :ref:`OAK-1 Lite`, :ref:`OAK-1 Lite W`, :ref:`OAK-1 Max`):

.. image:: https://user-images.githubusercontent.com/18037362/202103792-45ed20b4-4345-4af0-a266-1e2b0eb88d07.png

OAK-D-Lite FSIN
^^^^^^^^^^^^^^^

.. image:: https://user-images.githubusercontent.com/18037362/202106310-e3cbbaa8-2b22-41ae-95b7-ce06465bfbdc.png

Note that stereo camera pair and color cameras aren't connected together.

.. include::  /pages/includes/footer-short.rst
