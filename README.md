# Zapbox Controller Protocol

Zapbox controllers are standard Bluetooth Low Energy (BLE) peripherals.

![ZapboxControllers](https://user-images.githubusercontent.com/770066/218263675-09302b3f-201b-4fb5-a9ba-4f0a0156e816.jpg)

The controllers contain an analog thumbstick (with digital click) and an analog trigger, along with 4 additional digital inputs - A/B/Menu on the front and a grip input on the side.

There is also an Inertial Measurement Unit (IMU) containing an accelerometer and gyroscope. With the current firmware the IMU is configured with an output data rate of 200Hz.

The data is all exposed via a custom GATT service.

## Connection

After inserting batteries, press any button to wake the controller from sleep mode and begin BLE advertising. The LED flashes to indicate the controller is in advertising mode. The controller will return to sleep mode after advertising for 60 seconds.

The custom GATT service UUID is `121a0001-f6e4-4bb8-aec5-9e814ab54443`. This is part of the BLE advertising packets so the easiest way to scan for controllers is to add a filter for this service UUID.

Alternatively you can filter by name - the controllers are named `Zap ABCD` where ABCD is the last 4 hex digits of the Bluetooth address.

There is no need for any sort of pairing procedure with the controllers, you can just connect to them and perform service discovery to list the characteristics required to obtain the data. When the controller is connected the LED will stop flashing and switch to a continuous low brightness state.

Tip: You can connect to any BLE devices and explore the services and characteristics they expose using the **nRF Connect** app for [Android](https://play.google.com/store/apps/details?id=no.nordicsemi.android.mcp) or [iOS](https://apps.apple.com/gb/app/nrf-connect-for-mobile/id1054362403).

## Connection Interval

BLE devices only wake up to send or receive data briefly before sleeping until the next *connection interval*. The BLE specification allows the connection interval to range between 7.5ms and 4s. Although peripherals are able to request a particular interval, it is ultimately the other side of the connection (the `central` in BLE terminology) that decides the connection interval that will be used. In the BLE spec, connection intervals are specified as integers with units of 1.25ms; for example a connection interval of 12 represents 15ms.

Ideally we want the connection interval to be as short as possible to get the lowest possible latency for IMU and input data. Unfortunately most mobile devices don’t go all the way down to the theoretical 7.5ms minimum from the BLE spec.

iOS has an additional quirk - when initially connecting to a BLE peripheral the minimum connection interval that it will use is 30ms. It does allow the peripheral to issue a later request to reduce the connection interval down to 15ms, which it does then honour.

## BLE Characteristics

The BLE characteristics that are part of the custom service are all identified by UUID. Most of the bytes in the UUIDs are identical - it is just the second set of 4 characters that change - ie the `0001` in the Service UUID shown above is replaced with `0002` to obtain the full UUID of the Streaming Data Characteristic `121a0002-f6e4-4bb8-aec5-9e814ab54443`.

### `0002` - Streaming Data Characteristic

This is the main characteristic that provides data from the controller. It is set up to use *notifications* to pass data across, so you should enable notifications on this characteristic to receive updates. The format of the data is described in the next section.

### `0003` - Connection Interval Request

Due to the iOS quirk mentioned above, the peripheral needs to explicitly request a lower connection interval in order for iOS to choose it.

You can `WRITE` a desired connection interval to this characteristic. The firmware will then send a request to change the connection interval to that value to the central device. You can `READ` the info characteristic described below to determine if the request was accepted by the central.

On iOS the lowest accepted connection interval is 12 (or `0x0C`) for 15ms. My Pixel 4A accepts requests down to 9, giving a 11.25ms connection interval.

### `0004` - Info Characteristic

Doing a `READ` of this characteristic returns a 5 byte value:

- value[0] - Major Version (`0x01` in the preloaded firmware)
- value[1] - Minor Version (`0x07` in the preloaded firmware)
- value[2] - BLE MTU
- value[3] - BLE Data Length (PDU)
- value[4] - Connection Interval

The BLE specification by default allows only 27 bytes of data per packet, which is further reduced by the need for header information as part of the GATT protocols. That’s not a lot of data when we have multiple IMU readings to send per connection interval. Thankfully most mobile devices support a “Data Length Extension” that enables the peripheral and central to negotiate a higher value for the data length. The MTU and PDU data here provides some insight into what was negotiated for the current connection.

The final byte gives the current connection interval (in units of 1.25ms).

### `0005` - IMU Control

This characteristic allows the central to poke registers of the IMU to change its configuration without requiring a firmware update. Not currently used in our runtime and we’re not intending to document this any further.

Any changes to registers are only temporary - everything is reset to the default IMU config when waking from sleep.

## Data Format

Each GATT notification from the `0002` characteristic includes a fixed 8-byte header (including the input state and some timing fields) followed by a variable number of IMU samples (12 bytes per sample).

### Header Data

- header[0] - Digital button state bitmask and error flags
  - `0x01` A button
  - `0x02` B button
  - `0x04` Menu button
  - `0x08` Grip button
  - `0x10` Thumbstick click
  - `0b11100000` Error flags, not really needed
- header[1] - Analog trigger. Expected range roughly `0x2A` to `0x80`, but we recommend a per-controller calibration of min and max values as there is some variability in practice.
- header[2] - Thumbstick X. Roughly `0x80` in centre, minimum value when pushed **right**, maximum value when pushed **left**. We invert this in our runtime. We recommend calibrating zero point and maximum ranges on a per-controller basis.
- header[3] - Thumbstick Y. As above, but minimum when pushed down, maximum when pushed up.
- header[4] - IMU sample number (high byte)
- header[5] - IMU sample number (low byte)
- header[6] - Controller CPU RTC Counter (high byte)
- header[7] - Controller CPU RTC Counter (low byte)

The lower 16 bits of the controller CPU real-time clock counter were included in case it was helpful to model clock drift, but in practice with BLE the clock on the central determines the connection interval and the IMU has its own clock, so the controller CPU clock doesn’t really come into play and this field isn’t used in our runtime.

The IMU is running at 200 Hz, so you can convert from the IMU sample number to an “idealized” IMU sample time by multiplying by 1/200 seconds. Only the lower 16 bits of the sample count are included in the data so the client code should maintain a variable for the higher bits and increment that when the 16 bit sample count overflows.

Some example code for converting the sample number to a idealized IMU timestamp in microseconds is below:
```C
// Constant conversion factor for timestamp
int usPerImuSample = 1000000 / 200;

// These could be member variables if you want
static uint64_t prevImuSampleLowBits_ = 0;
static uint64_t imuSampleHighBits_ = 0;

// Given a BLE characteristic update message with uint8_t* data

// Get low 16 bits from the data, increment higher bits on overflow
uint16_t imuSampleLowBits = (uint16_t)((data[4] << 8) | data[5]);
if(imuSampleLowBits < prevImuSampleLowBits_) {
    imuSampleHighBits_ += (1 << 16);
}
prevImuSampleLowBits_ = imuSampleLowBits;

// Calculate the full sample number and idealized timestamp for the
// first IMU sample in this update notification
uint64_t imuSample = imuSampleHighBits_ + imuSampleLowBits;
uint64_t timestampUs = imuSample * usPerImuSample;
```

You will likely want to correlate the idealized IMU times with the clock on your host device. The data for the upcoming connection interval is prepared 4.5ms in advance of the connection interval, so assuming the final IMU sample is from roughly 5ms before the update is received is close enough for our current usage in the runtime. More accurate clock drift modelling would be possible but isn’t something we’ve needed to think about in detail yet.

### IMU sample data

After the 8-byte header there are a variable number of IMU samples, 12 bytes per sample. Samples are never split between characteristic notification updates, so you can always assume all the data for a single sample is in the same update message. In other words the length of the BLE update is always some integer multiple of 12 bytes + the 8 byte header.

Each 12-byte IMU sample contains the raw x, y, z accelerometer readings followed by the raw x, y, z gyroscope readings. Each value is a signed 16-bit integer in big endian byte ordering.

Example code to extract the raw values into int16_t variables is shown below:

```C

// Assuming the uint8_t* data, as in the previous code example
for (int idx = 8; idx<dataLen-11; idx+=12) {
    int16_t accelData[3];
    accelData[0] = (int16_t)((data[idx+0] << 8) | data[idx+1]);
    accelData[1] = (int16_t)((data[idx+2] << 8) | data[idx+3]);
    accelData[2] = (int16_t)((data[idx+4] << 8) | data[idx+5]);
    
    int16_t gyroData[3];
    gyroData[0] = (int16_t)((data[idx+6] << 8) | data[idx+7]);
    gyroData[1] = (int16_t)((data[idx+8] << 8) | data[idx+9]);
    gyroData[2] = (int16_t)((data[idx+10] << 8) | data[idx+11]);
    
    uint64_t timestampUs = imuSample * usPerImuSample;
    
    // Do whatever processing you want
    processSample(timestampUs, accelData, gyroData);
    
    imuSample++;
}
```

The accelerometer measures the current acceleration the controller is being subjected to due to the forces being applied to it. The values are signed 16-bit integers with full-scale (+/- 32768) representing +/- 16g of acceleration.

Note that at rest on a surface there is a force applied to the controller by the surface to counteract gravity. Therefore at rest on a solid surface the IMU should be expected to report an acceleration of 1g in the upwards direction (away from gravity). With the 16g full scale that implies you should expect a reading of around 2048 in the upwards direction in the raw data when the controller is stationary.

The IMU is located just above the menu button on the PCB. With the controller held vertically with the tracking marker at the top and the thumbstick facing you, the IMU +X axis is downwards, +Y is to the right, and +Z is pointing towards you, as shown in the image below.

<img width="339" alt="IMU Position and Axes" src="https://user-images.githubusercontent.com/770066/220760207-71168734-2623-4a76-b7b1-17dcb4a68337.png">

The gyroscope measures the speed (or rate) of rotation around the 3 axes. The full-scale range (+/- 32768) represents +/- 2000 degrees per second. When the controller is stationary the gyro readings should be zero. The sensor may not report exactly 0 when stationary due to gyro bias - we recommend calibrating for bias by averaging readings over a small set of samples when the controller is stationary (the range of the samples will be small during stationary periods) and then subtracting this bias estimate from the raw measurements before any further processing of the data.

The sense of the rotation follows the “right hand screw rule” - if you point your right thumb along the +ve axis, then positive rotations curl around in the direction of your fingers. The orange arrows on the image above show the +ve rotation directions for the gyro measurements.
