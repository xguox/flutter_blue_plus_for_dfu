
# migration guide

Breaking changes in FlutterBluePlus, listed version by version.

## 1.10.0

### .instance removed

You no longer need to use `.instance`

i.e. `FlutterBluePlus.instance.startScan` becomes `FlutterBluePlus.startScan`

### turnOn and turnOff

Typically no code changes are required. However:

* they now properly wait for completion if you use `await`
* they throw on error, instead of return true & false

## 1.15.0

### `FlutterBluePlus.scan` was removed

**Option 1:** migrate to `FlutterBluePlus.startScan` with `oneByOne` parameter

**Option 2:** use the following extension (below)

```
extension Scan on FlutterBluePlus {
  static Stream<ScanResult> scan({
    List<Guid> withServices = const [],
    Duration? timeout,
    bool androidUsesFineLocation = false,
  }) {
    if (FlutterBluePlus.isScanningNow) {
        throw Exception("Another scan is already in progress");
    }

    final controller = StreamController<ScanResult>();

    var subscription = FlutterBluePlus.scanResults.listen(
      (r) => controller.add(r.first),
      onError: (e, stackTrace) => controller.addError(e, stackTrace),
    );

    FlutterBluePlus.startScan(
      withServices: withServices,
      timeout: timeout,
      removeIfGone: null,
      oneByOne: true,
      androidUsesFineLocation: androidUsesFineLocation,
    );

    Future scanComplete = FlutterBluePlus.isScanning.where((e) => e == false).first;

    scanComplete.whenComplete(() {
      subscription.cancel();
      controller.close();
    });

    return controller.stream;
  }
}
```

---

### `FlutterBluePlus.startScan` doesn't return List<ScanResult> anymore

**Option 1:** migrate to `FlutterBluePlus.startScan`. Example code:

```
Stream<BluetoothDevice?> myDeviceStream = FlutterBluePlus.scanResults
    .map((list) => list.first)
    .where((r) => r.device.localName == "myDeviceName")
    .map((r) => r.device);

// start listening before we call startScan so we do not miss the result
Future<BluetoothDevice?> myDeviceFuture = myDeviceStream.first
    .timeout(Duration(seconds: 10))
    .catchError((error) => null);

await FlutterBluePlus.startScan(timeout: Duration(seconds: 10), oneByOne:true);

BluetoothDevice? myDevice = await myDeviceFuture;
```

**Option 2:** use this extension

```
extension Scan on FlutterBluePlus {
  static Future<List<ScanResult>> startScanWithResult({
    List<Guid> withServices = const [],
    Duration? timeout,
    bool androidUsesFineLocation = false,
  }) async {
    if (FlutterBluePlus.isScanningNow) {
      throw Exception("Another scan is already in progress");
    }

    List<ScanResult> output = [];

    var subscription = FlutterBluePlus.scanResults.listen((result) {
      output = result;
    }, onError: (e, stackTrace) {
      throw Exception(e);
    });

    FlutterBluePlus.startScan(
      withServices: withServices,
      timeout: timeout,
      removeIfGone: null,
      oneByOne: false,
      androidUsesFineLocation: androidUsesFineLocation,
    );

    // wait scan complete
    await FlutterBluePlus.isScanning.where((e) => e == false).first;

    subscription.cancel();

    return output;
  }
}
```

---

### `await FlutterBluePlus.startScan()` does not wait for scan completion anymore

Use `isScanning` to detect completion instead.

```
await FlutterBluePlus.startScan(timeout: Duration(seconds:15));
await FlutterBluePlus.isScanning.where((value) => value == false).first;
```

### `FlutterBluePlus.startScan` doesn't support macAddress filtering anymore

You can easily filter by mac address yourself. 

```
Stream<BluetoothDevice?> myDeviceStream = FlutterBluePlus.scanResults
    .map((list) => list.where(device) => device.remoteId == Guid("be6e0363-906a-4af6-9417-4b6085eb2e94"))
    .where((list) => list.isNotEmpty)
    .map((list) => list.first)

// start listening before we call startScan so we do not miss the result
Future<BluetoothDevice?> myDeviceFuture = myDeviceStream.first
    .timeout(Duration(seconds: 10))
    .catchError((error) => null);

await FlutterBluePlus.startScan(timeout: Duration(seconds: 10));

BluetoothDevice? myDevice = await myDeviceFuture;
```