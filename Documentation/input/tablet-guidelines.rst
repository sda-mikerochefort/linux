.. SPDX-License-Identifier: GPL-2.0

=========================================
Graphics tablet and pen display interface
=========================================

This document describes the evdev interface that kernel drivers and HID-BPF
programs must expose for graphics tablets, pen displays and tablet "remotes"
(standalone pads). It supplements the tablet section of
Documentation/input/event-codes.rst and Documentation/input/multi-touch-protocol.rst.

The goal is that every tablet presents the same evdev topology regardless of
whether it is handled by ``hid-generic`` plus a HID-BPF program, by ``wacom``
or by ``hid-uclogic``. Userspace (udev, libinput, libwacom) then needs no
vendor-specific heuristics.

The keywords MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are to be interpreted
as described in RFC 2119.

Terminology
===========

pen tablet
  An opaque digitizer whose coordinates are mapped to a screen through an
  on-screen pointer (for example Wacom Intuos, XP-Pen Deco, Huion Inspiroy).

pen display
  A digitizer integrated with, or laid over, a screen (for example Wacom
  Cintiq, XP-Pen Artist, Huion Kamvas), including the pen digitizer of a
  convertible laptop.

tool
  A stylus, eraser end, airbrush, puck/lens cursor or any other object
  tracked by the digitizer.

pad
  Buttons, touch rings, touch strips and rotary dials that are part of the
  tablet but not of the tool ("ExpressKeys").

remote / standalone pad
  A pad without its own digitizer, for example a wireless shortcut remote.

Device topology
===============

A physical device MUST expose at most one evdev node per function:

=========  ==============================  ==============================
Node       Present on                      Contents
=========  ==============================  ==============================
Stylus     every tablet and pen display    all tools
Pad        devices with pad controls       buttons, rings, strips, dials
Touch      devices with finger touch       multi-touch, see below
=========  ==============================  ==============================

All pad controls of one physical device MUST be reported on a single pad node.
Rings, strips and dials MUST NOT be split onto separate evdev nodes.

Vendor interfaces that only emulate a keyboard or mouse for the pad or tool
(commonly used by vendors as a fallback for systems without a driver) MUST NOT
produce additional evdev nodes once the device is in its native tablet mode.
A driver or HID-BPF program that switches the device into its native mode
SHOULD also suppress the emulation collections, for example by rewriting the
report descriptor to drop them.

All nodes of a physical device MUST be children of the same HID device or, for
devices with several HID interfaces, of the same USB (or Bluetooth) device, so
that userspace can group them by topology.

Device names SHOULD end in ``Pen``, ``Pad`` and ``Touch`` (or ``Finger``)
respectively, following the convention used by ``hid-input`` and ``wacom``.

Identification and reused product IDs
-------------------------------------

Some vendors reuse USB vendor/product IDs for unrelated hardware. Where the
hardware provides a firmware or model identifier that distinguishes such
devices, the driver or HID-BPF program SHOULD store it in ``hid_device.uniq``
so that it becomes ``EVIOCGUNIQ`` and the ``uniq`` sysfs attribute of every
evdev node of that device. HID-BPF programs attached through
``struct hid_bpf_ops`` may write ``hid_device.uniq`` directly. The value MUST be
stable across reboots and MUST NOT contain per-unit serial numbers unless no
model identifier is available.

Stylus node
===========

Properties
----------

A pen display MUST set ``INPUT_PROP_DIRECT`` and MUST NOT set
``INPUT_PROP_POINTER``. A pen tablet MUST set ``INPUT_PROP_POINTER`` and MUST
NOT set ``INPUT_PROP_DIRECT``.

``hid-input`` derives this property from the HID application collection
(Digitizer: pointer, Pen: direct). Devices that declare the wrong application
collection MUST be fixed, either in their driver or by a HID-BPF report
descriptor fixup.

Axes
----

=====================  ==========  ==================================================
Code                   Status      Semantics
=====================  ==========  ==================================================
``ABS_X``, ``ABS_Y``   required    Tool position. ``resolution`` MUST be set, in units
                                   per millimetre, and MUST describe the physical
                                   size of the active area.
``ABS_PRESSURE``       required    Tip pressure. Minimum 0. MUST be 0 while the tip
                                   is not in contact. ``resolution`` is unused.
``ABS_TILT_X``,        if present  Tilt as defined by the HID Digitizer page usages
``ABS_TILT_Y``                     X Tilt and Y Tilt. 0 is perpendicular to the
                                   surface. ``resolution`` SHOULD be set, in units
                                   per radian (``hid-input`` converts degrees).
``ABS_DISTANCE``       if present  Hover distance. Larger values are further away.
``ABS_Z``              if present  Barrel rotation of the tool around its own axis.
                                   ``resolution`` SHOULD be set, in units per radian.
``ABS_WHEEL``          if present  Airbrush finger wheel or slider. Note that
                                   ``ABS_WHEEL`` has a different meaning on the pad
                                   node.
``ABS_MISC``           if present  Hardware tool ID (tool type), see below.
=====================  ==========  ==================================================

Axes the hardware cannot measure MUST NOT be advertised.

Tools and proximity
-------------------

Every tool type the hardware can distinguish MUST be advertised with the
matching ``BTN_TOOL_*`` code: ``BTN_TOOL_PEN``, ``BTN_TOOL_RUBBER``,
``BTN_TOOL_BRUSH``, ``BTN_TOOL_PENCIL``, ``BTN_TOOL_AIRBRUSH``,
``BTN_TOOL_MOUSE`` or ``BTN_TOOL_LENS``.

- When a tool enters proximity, the driver MUST set its ``BTN_TOOL_*`` code to
  1 in the same frame as its first position. When it leaves proximity, the
  driver MUST set it to 0 and MUST NOT report further axis changes for that
  tool.
- At most one ``BTN_TOOL_*`` code MAY be set at any time. To switch from the
  pen tip to the eraser (or vice versa), the driver MUST first send a frame
  taking the old tool out of proximity, then a frame bringing the new tool
  into proximity.
- Pens whose "eraser" is a side button that the firmware reports as a separate
  bit or chord SHOULD be converted to a ``BTN_TOOL_RUBBER`` proximity transition
  in the driver or HID-BPF program where this can be done without guessing.
  Otherwise the raw button MUST be reported as a stylus button.
- ``BTN_TOUCH`` MUST be 1 while the tip (or eraser) is in contact with the
  surface and 0 otherwise.

Tool identification
-------------------

If the hardware reports a per-tool serial number, it MUST be reported with
``MSC_SERIAL`` in every frame in which the tool is in proximity, and the
hardware tool type ID SHOULD be reported in ``ABS_MISC``. Drivers MUST NOT
invent serial numbers. If no hardware serial is available, ``MSC_SERIAL`` MUST
NOT be advertised. Userspace is responsible for synthesizing a stable tool
identity in that case.

Buttons
-------

Buttons on the tool MUST be reported as ``BTN_STYLUS``, ``BTN_STYLUS2`` and
``BTN_STYLUS3``, in that order, from the button closest to the tip outward.
The stylus node MUST NOT report pad buttons (``BTN_0`` and following),
keyboard keys (``KEY_*``), ``BTN_LEFT`` or ``BTN_MOUSE`` for pen tools, nor any
``EV_REL`` axis.

Puck/lens tools are the exception: their buttons are reported with
``BTN_LEFT``, ``BTN_RIGHT``, ``BTN_MIDDLE``, ``BTN_SIDE`` and ``BTN_EXTRA``, and
their wheel with ``REL_WHEEL``, as the ``wacom`` driver does today.

Pad node
========

Identification
--------------

udev identifies a pad node (``ID_INPUT_TABLET_PAD``) if either of the following
is true:

- the node has ``ABS_X``, ``ABS_Y``, ``BTN_STYLUS``, ``BTN_0`` and ``BTN_1``, and
  no ``BTN_TOOL_PEN``; or
- the node has ``BTN_0`` and ``BTN_1`` plus ``REL_WHEEL`` or ``REL_HWHEEL``, and
  no ``REL_X`` or ``REL_Y``.

A pad node MUST therefore satisfy one of these conditions, even if the pad has
fewer than two buttons. Pads that do not otherwise report relative axes MUST
use the first form. Its ``ABS_X`` and ``ABS_Y`` axes are placeholders, with a
range of 0 to 1. They MUST NOT change value. The ``wacom`` driver and existing
HID-BPF programs already follow this convention.

``BTN_A`` and the codes after it fall into the range udev counts as joystick
buttons. A pad identified only through the second form, with more than ten
buttons, is therefore also tagged ``ID_INPUT_JOYSTICK``. Such pads SHOULD use
the first form.

Buttons
-------

Pad button *n*, numbered from 0, MUST be reported with the following codes:

==========  ===============================================================
*n*         Code
==========  ===============================================================
0 – 9       ``BTN_0`` + *n*
10 – 15     ``BTN_A``, ``BTN_B``, ``BTN_C``, ``BTN_X``, ``BTN_Y``, ``BTN_Z``
16 – 17     ``BTN_BASE``, ``BTN_BASE2``
==========  ===============================================================

This is the numbering used by the ``wacom`` driver. Pads with more than 18
buttons are not currently supported by this interface. The physical location
and label of each button is described in userspace (libwacom), not by the
kernel. Mode-switch buttons are ordinary pad buttons.

Touch rings
-----------

The first ring MUST be reported as ``ABS_WHEEL`` and the second ring as
``ABS_THROTTLE``. The range covers one full revolution, with the minimum
at the leftmost (west) position, increasing clockwise.

Touch strips
------------

The first strip MUST be reported as ``ABS_RX`` and the second strip as
``ABS_RY``. A value of 0 is reserved and means "no finger". Positions MUST
use values from 1 to the maximum. The ``wacom`` driver reports strip positions
as single set bits (``1 << position``). New drivers SHOULD report a linear
position instead.

Rotary dials
------------

Rotary dials are relative controls. The first dial MUST be reported as
``REL_WHEEL`` and ``REL_WHEEL_HI_RES``. The second dial MUST be reported as
``REL_HWHEEL`` and ``REL_HWHEEL_HI_RES``. The high-resolution codes follow the
semantics in Documentation/input/event-codes.rst: 120 represents one detent.
Clockwise rotation, as seen by the user with the device in its default
orientation, MUST be reported as a positive value. Drivers MUST NOT report
``REL_DIAL`` together with ``REL_WHEEL`` on a pad node.

Finger lift on rings and strips
-------------------------------

While a finger is on a ring or strip, the driver SHOULD report a non-zero
``ABS_MISC`` value on the pad node. When the finger is lifted, it SHOULD reset
the ring or strip axis and report ``ABS_MISC`` 0 in the same frame. This lets
userspace tell a finger lift apart from a position. The ``wacom`` driver
reports ``ABS_MISC`` this way.

Touch node
==========

Finger touch on pen displays and on pen tablets with touch MUST be reported on
its own node, using the type B multi-touch protocol described in
Documentation/input/multi-touch-protocol.rst. The touch node of a pen display
MUST set ``INPUT_PROP_DIRECT``. ``ABS_MT_POSITION_X`` and ``ABS_MT_POSITION_Y``
MUST have a resolution set.

The touch node MUST NOT emit pointer-emulation events, such as a single-touch
``BTN_LEFT``, in addition to the multi-touch data. Single-touch ``ABS_X``,
``ABS_Y`` and ``BTN_TOUCH`` follow multi-touch-protocol.rst.

Arbitration between pen and touch is a userspace policy. New drivers SHOULD
NOT suppress touch while a pen is in proximity. The ``wacom`` driver does so
by default for compatibility (module parameter ``touch_arbitration``).

Mode LEDs
=========

LEDs that show the current mode of a pad mode group MUST be registered as LED
class devices on the pad's input device, one LED per mode, named::

  <input device>::wacom-<group>.<mode>

where ``<group>`` and ``<mode>`` are 0-based. This is the name that the
``wacom`` driver registers today and that userspace already consumes. The
``wacom_led`` sysfs attributes are legacy ABI and MUST NOT be added to other
drivers. RGB indicators SHOULD use the multicolor LED class.

.. note::

   The ``wacom-`` prefix is retained for compatibility with existing
   userspace. A vendor-neutral name is under discussion.

Battery
=======

Wireless tablets, pens with a battery and wireless remotes SHOULD report their
battery state through a ``power_supply`` device with
``POWER_SUPPLY_SCOPE_DEVICE``, parented to the HID device. Devices behind a
wireless receiver SHOULD keep their ``power_supply`` and input devices across
receiver connection and disconnection, as ``hid-logitech-hidpp`` does.

Auxiliary displays
==================

Some remotes and tablets include small displays for button labels. The kernel
MUST NOT render content for them. Such displays are driven by userspace over
``hidraw``. A compositor can obtain the ``hidraw`` node through
``systemd-logind``, which revokes access with ``HIDIOCREVOKE`` when the session
becomes inactive.

HID-BPF
=======

Where a device deviates from this document only in its report descriptor or
report format, the fix SHOULD be a HID-BPF program rather than a new kernel
driver. HID-BPF programs SHOULD check the report descriptor size and contents,
and SHOULD check any firmware identifier, before attaching. This prevents a
fixup from attaching to unrelated hardware that shares the same USB IDs.

Testing
=======

New devices and fixups SHOULD come with a test in
``tools/testing/selftests/hid/tests/test_tablet.py``, or the equivalent test
in the udev-hid-bpf project, that replays the device's report descriptor and
verifies the resulting evdev nodes against this document.
