# Flutter Blue Native Implementasyon Analizi

## Yönetici Özeti

Flutter Blue, Dart ve native kod (Android/iOS) arasında iletişim için sofistike bir mimari uygulayan cross-platform Bluetooth Low Energy eklentisidir. Eklenti, serialization için **Protocol Buffers (Protobuf)**, request-response pattern'leri için **Method Channels** ve veri akışı için **Event Channels** kullanır.

---

## 1. Mimari Genel Bakış

### 1.1 Üst Düzey İletişim Akışı

```
┌─────────────────────┐
│   Flutter Dart      │
│   (FlutterBlue)     │
│                     │
└──────────┬──────────┘
           │ Method Channels
           │ (Methods: startScan, connect, read, write, vb.)
           │ Event Channels
           │ (Streams: state, scan results, notifications)
           │
           ├─────────────────────┬──────────────────────┐
           │                     │                      │
      ┌────▼────┐           ┌────▼────┐         ┌─────▼─────┐
      │ Android │           │   iOS   │         │  Protobuf │
      │ Plugin  │           │ Plugin  │         │ Serializer│
      │         │           │         │         │           │
      └────┬────┘           └────┬────┘         └─────┬─────┘
           │                     │                     │
      ┌────▼────────────┐   ┌────▼─────────────┐      │
      │ BluetoothManager│   │ CBCentralManager  │      │
      │ BluetoothGatt   │   │ CBPeripheral      │      │
      │ (Android API)   │   │ (CoreBluetooth)   │      │
      └─────────────────┘   └───────────────────┘      │
                                                        │
                         Binary serialization ◄────────┘
```

### 1.2 Serileştirme Stratejisi

**Protocol Buffers (Protobuf)** tüm veri alışverişi için kullanılır:
- Tanımlı: `/protos/flutterblue.proto`
- Oluşturulan Dart kodu: `/lib/gen/flutterblue.pb.dart`
- Oluşturulan Java kodu: protoc-gen-javalite tarafından otomatik oluşturulur
- Oluşturulan Objective-C: `/ios/gen/Flutterblue.pbobjc.h/.m`

**Avantajları:**
- Dil-bağımsız binary format
- Kompakt gösterim
- Versiyon-toleranslı şema evrimi
- Derleme zamanında type-safe

---

## 2. Android Implementasyon Mimarisi

### 2.1 Ana Sınıf: FlutterBluePlugin

**Konum:** `/android/src/main/java/com/pauldemarco/flutter_blue/FlutterBluePlugin.java`

**Temel Özellikler:**
- Implements: `FlutterPlugin`, `ActivityAware`, `MethodCallHandler`, `RequestPermissionsResultListener`
- Hem V1 (Registrar-based) hem V2 (FlutterPluginBinding) embedding'i destekler
- Synchronized başlatma ile singleton pattern

**Kanal Kurulumu:**
```java
private static final String NAMESPACE = "plugins.pauldemarco.com/flutter_blue";
private MethodChannel channel;      // Methods için: startScan, connect, read, write
private EventChannel stateChannel;  // Bluetooth state stream için
```

### 2.2 Çekirdek Android Bileşenleri

**BluetoothManager & BluetoothAdapter:**
```java
mBluetoothManager = (BluetoothManager) application.getSystemService(Context.BLUETOOTH_SERVICE);
mBluetoothAdapter = mBluetoothManager.getAdapter();
```

**Cihaz Önbelleği:**
```java
private final Map<String, BluetoothDeviceCache> mDevices = new HashMap<>();

class BluetoothDeviceCache {
    final BluetoothGatt gatt;
    int mtu;  // Önbelleğe alınmış MTU boyutu (varsayılan 20)
}
```

### 2.3 Desteklenen Metodlar

Eklenti 17 method çağrısını işler:

| Method | İstek | Yanıt | Amaç |
|--------|---------|----------|---------|
| `setLogLevel` | LogLevel index | null | Loglama detayını ayarla |
| `state` | yok | BluetoothState proto | Mevcut BT durumunu al |
| `isAvailable` | yok | boolean | BT desteğini kontrol et |
| `isOn` | yok | boolean | BT'nin açık olup olmadığını kontrol et |
| `startScan` | ScanSettings proto | null | BLE taraması başlat |
| `stopScan` | yok | null | BLE taramasını durdur |
| `getConnectedDevices` | yok | ConnectedDevicesResponse proto | Bağlı cihazları listele |
| `connect` | ConnectRequest proto | null | Cihaza bağlan |
| `disconnect` | remoteId string | null | Cihaz bağlantısını kes |
| `deviceState` | remoteId string | DeviceStateResponse proto | Bağlantı durumunu al |
| `discoverServices` | remoteId string | null | Servisleri keşfet |
| `services` | remoteId string | DiscoverServicesResult proto | Keşfedilen servisleri al |
| `readCharacteristic` | ReadCharacteristicRequest proto | null | Characteristic oku |
| `readDescriptor` | ReadDescriptorRequest proto | null | Descriptor oku |
| `writeCharacteristic` | WriteCharacteristicRequest proto | null | Characteristic yaz |
| `writeDescriptor` | WriteDescriptorRequest proto | null | Descriptor yaz |
| `setNotification` | SetNotificationRequest proto | null | Notification'ları etkinleştir |
| `mtu` | remoteId string | MtuSizeResponse proto | Mevcut MTU'yu al |
| `requestMtu` | MtuSizeRequest proto | null | MTU değişikliği iste |

### 2.4 Asenkron Callback Mimarisi

**BluetoothGattCallback** (satırlar 857-985):
```java
private final BluetoothGattCallback mGattCallback = new BluetoothGattCallback() {
    // Bluetooth framework'ünden callback'ler:
    - onConnectionStateChange()      // Bağlantı durumu güncellemeleri
    - onServicesDiscovered()         // Servis keşfi tamamlandı
    - onCharacteristicRead()         // Okuma yanıtı
    - onCharacteristicWrite()        // Yazma onayı
    - onCharacteristicChanged()      // Notification'lar/Indication'lar
    - onDescriptorRead()             // Descriptor okuma yanıtı
    - onDescriptorWrite()            // Descriptor yazma onayı
    - onMtuChanged()                 // MTU müzakeresi tamamlandı
};
```

Tüm callback olayları Dart'a şu şekilde çağrı yapar:
```java
private void invokeMethodUIThread(final String name, final byte[] byteArray) {
    activity.runOnUiThread(() -> channel.invokeMethod(name, byteArray));
}
```

### 2.5 Bluetooth Durum Stream'i

**StreamHandler Pattern** (satırlar 698-739):
- `BluetoothAdapter.ACTION_STATE_CHANGED` için `BroadcastReceiver` kaydeder
- Durum değişikliklerini `EventSink` üzerinden Dart'a iletir
- Stream iptal edildiğinde receiver kaydını siler

### 2.6 Tarama Implementasyonu

**API Seviyesi Ayrımı:**
```java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    startScan21();  // BluetoothLeScanner kullanır (modern)
} else {
    startScan18();  // Deprecated startLeScan kullanır (legacy)
}
```

**Tekrar Önleme:**
```java
private ArrayList<String> macDeviceScanned = new ArrayList<>();
private boolean allowDuplicates = false;

// Callback içinde: if (!allowDuplicates && macDeviceScanned.contains(address)) return;
```

### 2.7 Advertising Data Ayrıştırma

**AdvertisementParser.java:**
- Ham BLE advertising verilerini ayrıştırır
- Çıkarılanlar: local name, TX power, manufacturer data, service data, service UUID'ler
- `ByteBuffer` ile `ByteOrder.LITTLE_ENDIAN` kullanır

**Ayrıştırılan Temel AD Tipleri:**
- 0x08/0x09: Local name (kısa/uzun)
- 0x0A: TX Power Level
- 0x16/0x20/0x21: Service Data (16/32/128-bit UUID)
- 0xFF: Manufacturer Specific Data

### 2.8 İzin İşleme

**Android Runtime İzinleri:**
```java
// BLE taraması için ACCESS_FINE_LOCATION gerekli (Android 6.0+)
if (ContextCompat.checkSelfPermission(activity, Manifest.permission.ACCESS_FINE_LOCATION)
    != PackageManager.PERMISSION_GRANTED) {
    // Bekleyen çağrı/sonucu kaydet, izin iste
    ActivityCompat.requestPermissions(activity,
        new String[]{Manifest.permission.ACCESS_FINE_LOCATION},
        REQUEST_FINE_LOCATION_PERMISSIONS);
}

// onRequestPermissionsResult() içinde devam et
@Override
public boolean onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults) {
    if (requestCode == REQUEST_FINE_LOCATION_PERMISSIONS) {
        if (grantResults[0] == PackageManager.PERMISSION_GRANTED) {
            startScan(pendingCall, pendingResult);
        }
    }
}
```

### 2.9 Proto Dönüşüm Yardımcıları

**ProtoMaker.java:**
Android nesnelerini Protobuf'lara dönüştürmek için static factory metodları içerir:
- `from(BluetoothDevice)` → `Protos.BluetoothDevice`
- `from(BluetoothGattService)` → `Protos.BluetoothService`
- `from(BluetoothGattCharacteristic)` → `Protos.BluetoothCharacteristic`
- `from(ScanResult)` → `Protos.ScanResult` (API 21+ ve legacy'yi işler)

---

## 3. iOS Implementasyon Mimarisi

### 3.1 Ana Sınıf: FlutterBluePlugin

**Konum:** `/ios/Classes/FlutterBluePlugin.m` (35KB, ~774 satır)

**Temel Özellikler:**
- Implements: `FlutterPlugin`, `CBCentralManagerDelegate`, `CBPeripheralDelegate`
- Yalnızca Core Bluetooth Framework kullanır
- `+registerWithRegistrar:` içinde singleton-style başlatma

**Kanal Kurulumu:**
```objective-c
#define NAMESPACE @"plugins.pauldemarco.com/flutter_blue"

FlutterMethodChannel *channel = [FlutterMethodChannel
    methodChannelWithName:NAMESPACE @"/methods"
    binaryMessenger:[registrar messenger]];

FlutterEventChannel *stateChannel = [FlutterEventChannel
    eventChannelWithName:NAMESPACE @"/state"
    binaryMessenger:[registrar messenger]];
```

### 3.2 Çekirdek iOS Bileşenleri

**Central Manager & Peripheral'lar:**
```objective-c
@property(nonatomic, retain) CBCentralManager *centralManager;
@property(nonatomic) NSMutableDictionary *scannedPeripherals;
```

**Servis & Characteristic Keşif Takibi:**
```objective-c
@property(nonatomic) NSMutableArray *servicesThatNeedDiscovered;
@property(nonatomic) NSMutableArray *characteristicsThatNeedDiscovered;
```

### 3.3 Desteklenen Metodlar

iOS implementasyonu Android ile aynı metodları destekler, platform-spesifik limitasyonlarla:

| Method | Android | iOS | Notlar |
|--------|---------|-----|--------|
| `state` | ✓ | ✓ | |
| `startScan` | ✓ | ✓ | iOS: allowDuplicates seçeneği destekler |
| `stopScan` | ✓ | ✓ | |
| `connect` | ✓ | ✓ | iOS: peripheral almak için identifier kullanır |
| `disconnect` | ✓ | ✓ | |
| `requestMtu` | ✓ | ✗ | iOS limitasyonu: MTU sistem tarafından belirlenir |
| Diğer tüm metodlar | ✓ | ✓ | |

### 3.4 Asenkron Callback Mimarisi

**CBCentralManagerDelegate** (satırlar 363-401):
```objective-c
- (void)centralManagerDidUpdateState:(CBCentralManager *)central;
- (void)centralManager:didDiscoverPeripheral:advertisementData:RSSI:;
- (void)centralManager:didConnectPeripheral:;
- (void)centralManager:didDisconnectPeripheral:error:;
- (void)centralManager:didFailToConnectPeripheral:error:;
```

**CBPeripheralDelegate** (satırlar 404-539):
```objective-c
- (void)peripheral:didDiscoverServices:;
- (void)peripheral:didDiscoverCharacteristicsForService:error:;
- (void)peripheral:didDiscoverDescriptorsForCharacteristic:error:;
- (void)peripheral:didUpdateValueForCharacteristic:error:;
- (void)peripheral:didWriteValueForCharacteristic:error:;
- (void)peripheral:didUpdateNotificationStateForCharacteristic:error:;
- (void)peripheral:didUpdateValueForDescriptor:error:;
- (void)peripheral:didWriteValueForDescriptor:error:;
```

### 3.5 Temel Tasarım Pattern'leri

**CBUUID Extension:**
```objective-c
@interface CBUUID (CBUUIDAdditionsFlutterBlue)
- (NSString *)fullUUIDString;
@end

// 16-bit UUID'leri tam 128-bit formata dönüştürür
// "180A" → "0000180a-0000-1000-8000-00805f9b34fb"
```

**Keşif State Machine:**
Eklenti servis ve characteristic keşfini takip etmek için yardımcı diziler kullanır:
1. `didDiscoverServices:` - Servisleri keşif kuyruğuna ekler, characteristic'leri keşfeder
2. `didDiscoverCharacteristicsForService:` - Characteristic'leri kuyruğa ekler, descriptor'ları keşfeder
3. `didDiscoverDescriptorsForCharacteristic:` - Kuyruktan çıkarır, tamamlanıp tamamlanmadığını kontrol eder
4. Hepsi keşfedildiğinde, `DiscoverServicesResult` gönderir

**MTU Belirleme:**
```objective-c
- (uint32_t)getMtu:(CBPeripheral *)peripheral {
    if (@available(iOS 9.0, *)) {
        return (uint32_t)[peripheral maximumWriteValueLengthForType:
            CBCharacteristicWriteWithoutResponse];
    } else {
        return 20;  // Minimum fallback
    }
}
```

### 3.6 Bluetooth Durum Stream'i

**FlutterBlueStreamHandler:**
```objective-c
@interface FlutterBlueStreamHandler : NSObject<FlutterStreamHandler>
@property FlutterEventSink sink;
@end

- (FlutterError*)onListenWithArguments:(id)arguments eventSink:(FlutterEventSink)eventSink {
    self.sink = eventSink;
    return nil;
}
```

Durum değişiklikleri şu şekilde yayılır:
```objective-c
- (void)centralManagerDidUpdateState:(CBCentralManager *)central {
    if(_stateStreamHandler.sink != nil) {
        self.stateStreamHandler.sink([self toFlutterData:result]);
    }
}
```

### 3.7 Peripheral Keşfi & Önbellekleme

**Peripheral Saklama:**
```objective-c
[self.scannedPeripherals setObject:peripheral
    forKey:[[peripheral identifier] UUIDString]];
```

**Peripheral Getirme:**
```objective-c
- (CBPeripheral*)findPeripheral:(NSString*)remoteId {
    NSArray<CBPeripheral*> *peripherals =
        [_centralManager retrievePeripheralsWithIdentifiers:
            @[[[NSUUID alloc] initWithUUIDString:remoteId]]];
    // Eşleşen UUID'yi ara
}
```

### 3.8 Proto Dönüşüm Metodları

Eklenti ~20 proto dönüşüm metodu içerir:
- `toBluetoothStateProto:`
- `toScanResultProto:advertisementData:RSSI:`
- `toDeviceProto:`
- `toServiceProto:service:`
- `toCharacteristicProto:characteristic:`
- `toDescriptorProto:descriptor:`
- `toCharacteristicPropsProto:`
- `toMtuSizeResponseProto:mtu:`
- `toFlutterData:` (proto'yu FlutterStandardTypedData olarak sarmalar)

---

## 4. Platform Kanal Implementasyonu

### 4.1 Method Channel Mimarisi

**Namespace:** `plugins.pauldemarco.com/flutter_blue/methods`

**İletişim Pattern'i:**
```
Dart → Method Channel → Native Platform
Native Platform → invoke method → Dart
```

**Dart Tarafı (lib/src/flutter_blue.dart):**
```dart
final MethodChannel _channel =
    const MethodChannel('$NAMESPACE/methods');

// Native metod çağırma
await _channel.invokeMethod('startScan', settings.writeToBuffer());

// Native'den çağrıları alma
_channel.setMethodCallHandler((MethodCall call) {
    _methodStreamController.add(call);
    return;
});
```

**Native → Dart Çağrıları:**
Tüm olaylar `invokeMethod` üzerinden gönderilir:
- `ScanResult` - Yeni cihaz keşfedildi
- `DeviceState` - Bağlantı durumu değişti
- `DiscoverServicesResult` - Servisler keşfedildi
- `ReadCharacteristicResponse` - Okuma tamamlandı
- `WriteCharacteristicResponse` - Yazma tamamlandı
- `OnCharacteristicChanged` - Notification alındı
- `ReadDescriptorResponse` - Descriptor okundu
- `WriteDescriptorResponse` - Descriptor yazıldı
- `SetNotificationResponse` - Notification etkinleştirildi
- `MtuSize` - MTU müzakere edildi

### 4.2 Event Channel Mimarisi

**Namespace:** `plugins.pauldemarco.com/flutter_blue/state`

**Akış:**
```
Native Bluetooth Durum Değişikliği → Event Channel → Dart Stream
```

**Pattern:**
- Android: `BroadcastReceiver` + `StreamHandler`
- iOS: `CBCentralManagerDelegate.centralManagerDidUpdateState:` + `FlutterBlueStreamHandler`

---

## 5. Veri Akışı Örnekleri

### 5.1 Tarama Akışı

```
Dart:
  scan() → writeToBuffer() → channel.invokeMethod('startScan', buffer)
           ↓
Android:
  onMethodCall() → startScan(ScanSettings)
  → ScanSettings oluştur, BluetoothLeScanner al
  → scanForPeripheralsWithServices()
  → ScanCallback.onScanResult() [asenkron]
  → ProtoMaker.from(device, result)
  → invokeMethodUIThread("ScanResult", byteArray)
           ↓
Dart:
  _methodStream.where(m => m.method == "ScanResult")
  → map(buffer → ScanResult.fromProto)
  → _scanResults.add(result)
```

### 5.2 GATT İşlem Akışı (Characteristic Okuma)

```
Dart:
  read() → ReadCharacteristicRequest proto
  → channel.invokeMethod('readCharacteristic', buffer)
           ↓
Android:
  onMethodCall() → locateGatt(), locateCharacteristic()
  → gatt.readCharacteristic(characteristic)  [hemen true/false döner]
           ↓
  BluetoothGattCallback.onCharacteristicRead() [async callback]
  → ReadCharacteristicResponse proto oluştur
  → invokeMethodUIThread("ReadCharacteristicResponse", buffer)
           ↓
Dart:
  _methodStream.where(m => m.method == "ReadCharacteristicResponse")
  → map buffer → characteristic değeri
  → listener'a yayınla
```

### 5.3 Notification Kurulum Akışı

```
Dart:
  setNotifyValue(true) → SetNotificationRequest proto
           ↓
Android:
  setNotification() → locateCharacteristic()
  → gatt.setCharacteristicNotification(char, true)
  → CCCD descriptor al → gatt.writeDescriptor(CCCD)
           ↓
  BluetoothGattCallback.onDescriptorWrite()
  → CCCD UUID kontrolü → SetNotificationResponse gönder

  BluetoothGattCallback.onCharacteristicChanged() [async notification'lar]
  → OnCharacteristicChanged proto oluştur
  → invokeMethodUIThread("OnCharacteristicChanged", buffer)
```

---

## 6. Önemli Pattern'ler ve Mimari Kararlar

### 6.1 Serileştirme Kontratı Olarak Protobuf

**Neden Protobuf?**
- Diller arası tutarlılık sağlar (Java, Objective-C, Dart)
- Binary format, kanal iletişimi için veri boyutunu azaltır
- Geriye dönük uyumlu şema evrimi
- Derleme zamanında type safety

**Tek Doğruluk Kaynağı:**
`/protos/flutterblue.proto` - Üç implementasyon da aynı şemaya referans verir

### 6.2 Cihaz Önbellekleme Stratejisi

**Android:**
```java
Map<String, BluetoothDeviceCache> mDevices
// Saklar: GATT instance + MTU boyutu
```

**Amaç:**
- Aynı cihaz için GATT bağlantısını yeniden oluşturmaktan kaçın
- Daha önce bağlanmış cihazlara yeniden bağlanmayı destekle
- MTU değerini önbelleğe al (framework her zaman döndürmez)

### 6.3 UI Thread Marshalling (Android)

```java
private void invokeMethodUIThread(final String name, final byte[] byteArray) {
    activity.runOnUiThread(() -> channel.invokeMethod(name, byteArray));
}
```

**Neden:** Bluetooth callback'leri rastgele thread'lerde çalışır; kanal işlemleri ana thread'de olmalıdır.

### 6.4 Senkron-Asenkron Köprüsü

**Zorluk:** Bluetooth işlemleri async, ancak bazıları Dart'tan senkron başlatılır.

**Çözüm:**
- İstek metodları (`startScan`, `connect`, `read`) hemen döner (değer yok)
- Sonuçlar Dart'tan ayrı method çağrıları yoluyla gelir
- Framework çağrıların sırasını garanti eder

**Pattern:**
```
Dart Await
    ↓
Android başarı/hata döner
    ↓
(Bir yerlerde async...)
    ↓
invokeMethod ile sonuç callback'i
    ↓
Dart Stream
```

### 6.5 Servis Keşif State Machine (iOS)

iOS "keşif tamamlandı" callback'i sağlamaz. Bunun yerine:
- Keşfedilmesi gereken servisleri/characteristic'leri takip et
- Her descriptor keşfinden sonra, tamamlanıp tamamlanmadığını kontrol et
- Tüm kuyruklar boş olduğunda `DiscoverServicesResult` gönder

**Avantaj:** Tüm keşfi tek sonuçta toplamaya izin verir
**Trade-off:** Durum takibinde hafif karmaşıklık

### 6.6 Tekrarlanan Tarama Sonucu Filtreleme

**Varsayılan Davranış:** Tekrarlanan cihazları filtrele (aynı MAC adresi)
```java
allowDuplicates = settings.getAllowDuplicates();
if (!allowDuplicates && macDeviceScanned.contains(address)) return;
```

**İzin Verir:** `ScanSettings.allow_duplicates` ile filtreyi kontrol etme

### 6.7 İkincil Servis Desteği

Her iki platform da dahil edilen (ikincil) servisleri destekler:

**Yapı:**
```
Birincil Servis
  ├── Characteristic'ler
  └── Dahil Edilen Servisler (İkincil)
      └── Characteristic'ler
```

**Proto Saklama:**
```protobuf
message BluetoothCharacteristic {
    string serviceUuid;           // Birincil servis UUID
    string secondaryServiceUuid;  // İkincil servis UUID (iç içe ise)
}
```

### 6.8 CCCD (Client Characteristic Configuration Descriptor) İşleme

**Standart UUID:** `00002902-0000-1000-8000-00805f9b34fb`

**Manuel Yönetim Gerekli:**
- CCCD descriptor'ını bul
- Uygun değeri yaz (`ENABLE_NOTIFICATION`, `ENABLE_INDICATION`, `DISABLE`)
- Android: Hem descriptor yazma + characteristic notification etkinleştirme
- iOS: Otomatik; framework CCCD yazımını işler

### 6.9 API Seviyesi Soyutlama

**Android:**
```java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    // BluetoothLeScanner kullan (modern API 21+)
} else {
    // Deprecated startLeScan kullan (API <21)
}
```

**Destekler:** API 19+ (manifest'te minSdk 19 gerektirir)

---

## 7. Sorunlar ve Mimari Limitasyonlar

### 7.1 iOS-Spesifik Limitasyonlar

**requestMtu desteklenmiyor:**
```objective-c
} else if([@"requestMtu" isEqualToString:call.method]) {
    result([FlutterError errorWithCode:@"requestMtu"
        message:@"iOS does not allow mtu requests to the peripheral"
        details:NULL]);
```

Sebep: iOS, bağlantı müzakeresine göre MTU'yu belirler; açık istek API'si yok.

### 7.2 İzin Gecikmeleri (Android)

**Sorun:** İzin kontrolü gerekiyorsa `startScan` gecikebilir
```java
// Bekleyen çağrı/sonucu kaydet
pendingCall = call;
pendingResult = result;
// İzin iste
ActivityCompat.requestPermissions(activity, ...);
// onRequestPermissionsResult içinde devam et
```

**Trade-off:** İzin akışını otomatik yapar ancak async doğasını çağırandan gizler.

### 7.3 Cihaz Arama Verimsizliği (iOS)

```objective-c
- (CBPeripheral*)findPeripheral:(NSString*)remoteId {
    NSArray<CBPeripheral*> *peripherals =
        [_centralManager retrievePeripheralsWithIdentifiers:@[...]];
    for(CBPeripheral *p in peripherals) {
        if([[p.identifier UUIDString] isEqualToString:remoteId]) {
```

**Sorun:** Peripheral'lar içinde linear arama
**Daha İyi:** UUID'yi doğrudan kullan veya mapping önbelleğe al

### 7.4 Tarama Sonucu Tekrar Giderme (Android)

```java
private ArrayList<String> macDeviceScanned = new ArrayList<>();
```

**Sorun:** ArrayList O(n) arama süresine sahip
**Daha İyi:** O(1) arama için `HashSet<String>` kullan

### 7.5 Characteristic Toplu İşlem Yok

Her okuma/yazma işlemi ayrı callback tetikler:
```java
gatt.readCharacteristic(characteristic);  // Async
// onCharacteristicRead() HER characteristic için ayrı ayrı çağrılır
```

**Limitasyon:** Toplu okuma/yazma işlemleri yok; Dart'ta manuel olarak sıralanmalı

### 7.6 MTU Başlatma Tutarsızlığı

**Android:** Varsayılan 20, `onMtuChanged()` callback'inde güncellenir
**iOS:** Doğrudan sorgulanır, callback takibi yok

**Risk:** Android'de ilk callback'e kadar MTU değeri eski olabilir

### 7.7 İkincil Servis Keşfi

**iOS kodunda yorum (satır 417):**
```objective-c
// [peripheral discoverIncludedServices:nil forService:s];
// Secondary services in the future (#8)
```

**Sorun:** İkincil servisler iOS'ta tam olarak implemente edilmemiş
**Durum:** Eksik özellik

---

## 8. İletişim Kanalı Detayları

### 8.1 Method Channel Çağrı İmzası

**Android:**
```java
@Override
public void onMethodCall(MethodCall call, Result result) {
    String methodName = call.method;
    byte[] arguments = call.arguments();
    result.success(byteArray);  // veya error()
}
```

**iOS:**
```objective-c
- (void)handleMethodCall:(FlutterMethodCall*)call result:(FlutterResult)result {
    NSString *method = call.method;
    FlutterStandardTypedData *data = [call arguments];
    result(data);  // veya error
}
```

### 8.2 Hata İşleme

**Android:**
```java
result.error(errorCode, errorMessage, errorDetails);
// Örnek: result.error("connect", "Peripheral not found", nil);
```

**iOS:**
```objective-c
result([FlutterError errorWithCode:@"connect"
                           message:@"Peripheral not found"
                           details:nil]);
```

**Dart:**
```dart
try {
    await _channel.invokeMethod('method', args);
} on PlatformException catch (e) {
    // Hata işle
}
```

### 8.3 Binary Veri Kodlama

**Tüm payload'lar:**
- Serileştirilmiş Protobuf mesajı → `byte[]`
- Method/event kanalları üzerinden iletilir
- Dart tarafında deserialize edilir: `ProtoClass.fromBuffer(buffer)`

**Verimlilik:**
- Protobuf binary format JSON'dan ~%40-60 daha küçük
- İç içe yapıları natively destekler
- Tip bilgisi veride değil şemada kodlanır

---

## 9. Temel İmplementasyon İstatistikleri

| Metrik | Android | iOS |
|--------|---------|-----|
| Ana Plugin Satırları | 1,022 | 774 |
| Yardımcı Sınıflar | 2 (ProtoMaker, AdvertisementParser) | 0 |
| Desteklenen Bluetooth API'leri | Birden fazla (API 19+) | 1 (CoreBluetooth) |
| GATT Callback'leri | 8 | 8 |
| Tarama Modları | 4 + Legacy desteği | Native CBCentralManager |
| Proto Dönüşüm Metodları | 8 | 20+ |

---

## 10. Mimari Güçlü Yönlerin Özeti

1. **Dil-Bağımsız Serileştirme:** Protobuf tutarlılığı sağlar
2. **Çift-Kanal Pattern'i:** Farklı iletişim ihtiyaçları için Method + Event kanalları
3. **Callback Marshalling:** Uygun thread güvenliği (Android UI thread, iOS main thread)
4. **Cihaz Önbellekleme:** Gereksiz nesne oluşturmayı önler
5. **Versiyon Desteği:** Eski Android versiyonlarını zarif bir şekilde işler (API 19+)
6. **Type Safety:** Proto-generated kod derleme zamanı güvenliği sağlar
7. **Streaming Desteği:** Reactive stream'ler için RxDart entegrasyonu

---

## 11. İyileştirme Önerileri

1. **Tarama tekrar giderme için HashSet kullan (Android)**
2. **UUID ile peripheral arama önbelleği (iOS)**
3. **İkincil servis keşfini implemente et (iOS)**
4. **Toplu GATT işlemlerini destekle** (okuma/yazma toplu işlemleri)
5. **MTU durum senkronizasyonu ekle** (daha iyi takip)
6. **Async davranışı belgele** (method çağrıları vs sonuçlar)
7. **Uzun süren GATT işlemleri için timeout işleme ekle**
8. **Birden fazla eşzamanlı taramayı destekle** (şu anda özel)
