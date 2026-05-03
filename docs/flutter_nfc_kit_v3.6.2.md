# flutter_nfc_kit v3.6.2 — Knowledge Base

## ภาพรวม

`flutter_nfc_kit` คือ Flutter plugin สำหรับอ่าน/เขียน NFC Tags และ Smart Cards รองรับ 3 แพลตฟอร์ม:

| แพลตฟอร์ม | กลไกที่ใช้ | หมายเหตุ |
|---|---|---|
| Android | Android NFC API | รองรับครบทุกฟีเจอร์ |
| iOS | Apple CoreNFC | iOS 13+ เท่านั้น |
| Web | **WebUSB** (ไม่ใช่ WebNFC) | ต้องใช้ USB NFC Reader |

> ⚠️ **Web:** plugin นี้ใช้ WebUSB — ไม่ใช่ Web NFC API มาตรฐาน (chrome://flags/#enable-web-nfc) ดังนั้นต้องมี USB NFC Reader เชื่อมต่ออยู่

ใช้ร่วมกับ package `ndef` สำหรับ encode/decode NDEF Records

---

## ความสามารถหลัก

### 1. อ่าน/เขียน NDEF Records
รองรับแท็กมาตรฐาน:
- ISO 14443-A (MIFARE Classic, Ultralight, NTAG)
- ISO 14443-B
- ISO 18092 FeliCa
- ISO 15693 (RFID)

### 2. อ่าน/เขียน Block / Page / Sector (Raw Memory)
- MIFARE Classic — อ่าน/เขียนทีละ Sector (16 bytes)
- MIFARE Ultralight / NTAG — อ่าน/เขียนทีละ Page (4 bytes)

### 3. ส่งคำสั่ง Raw (APDU ISO 7816)
ใช้สำหรับ Smart Card เช่น EMV (บัตรเครดิต), ePassport, บัตรประชาชนไทย

---

## Setup Requirements

### Android

**Minimum Requirements:**
| รายการ | เวอร์ชันขั้นต่ำ |
|---|---|
| Java | 17 |
| Gradle | 8.9 |
| Android SDK | 24+ (Android 7.0) |
| Android Gradle Plugin | 8.7 |

**Permission ใน `AndroidManifest.xml`:**
```xml
<uses-permission android:name="android.permission.NFC" />
```

**Feature declaration (แนะนำ):**
```xml
<uses-feature android:name="android.hardware.nfc" android:required="true" />
```

---

### iOS

**Minimum Requirements:**
- iOS 13.0+
- Xcode Capability: **"Near Field Communication Tag Reading"** (ต้องเปิดใน Signing & Capabilities)

**Info.plist keys ที่ต้องใส่:**
```xml
<key>NFCReaderUsageDescription</key>
<string>แอปนี้ใช้ NFC เพื่ออ่านข้อมูลจากแท็กและการ์ด</string>

<key>com.apple.developer.nfc.readersession.felica.systemcodes</key>
<array>
    <string>88B4</string>
</array>

<key>com.apple.developer.nfc.readersession.iso7816.select-identifiers</key>
<array>
    <string>A0000002471001</string>
</array>
```

> ⚠️ **iOS Bug สำคัญ:** หากไม่ใส่ค่า `felica.systemcodes` และ `iso7816.select-identifiers` ก่อนเรียก `FlutterNfcKit.poll()` iOS CoreNFC จะ **ค้างจนต้อง reboot เครื่อง** นี่คือ bug ของ iOS CoreNFC ที่ยังไม่ได้รับการแก้ไข ต้องตรวจสอบ Info.plist ทุกครั้งก่อน build

---

### Web

ใช้ **WebUSB** — ต้องเชื่อมต่อ USB NFC Reader (เช่น ACS ACR122U)

```html
<!-- ไม่ต้องเพิ่ม permission พิเศษ แต่ browser ต้องรองรับ WebUSB -->
<!-- รองรับ: Chrome 61+, Edge 79+ -->
<!-- ไม่รองรับ: Firefox, Safari -->
```

---

## การติดตั้ง

### 1. เพิ่ม dependency ใน `pubspec.yaml`
```yaml
dependencies:
  flutter_nfc_kit: ^3.6.2
  ndef: ^0.3.1  # สำหรับ NDEF encoding/decoding
```

### 2. ติดตั้ง
```bash
flutter pub get
```

---

## Code Examples

### ตัวอย่างที่ 1: อ่านแท็ก NFC พื้นฐาน

```dart
import 'package:flutter_nfc_kit/flutter_nfc_kit.dart';
import 'package:ndef/ndef.dart' as ndef;

Future<void> readNfcTag() async {
  // ตรวจสอบว่าอุปกรณ์รองรับ NFC
  final availability = await FlutterNfcKit.nfcAvailability;
  if (availability != NFCAvailability.available) {
    print('NFC ไม่พร้อมใช้งาน: $availability');
    return;
  }

  try {
    // เริ่มรอแท็ก NFC (Android: ลดความหน่วง ~500ms, ปิดเสียง)
    const flags = 0x80 | 0x100; // FLAG_READER_SKIP_NDEF_CHECK | FLAG_READER_NO_PLATFORM_SOUNDS
    final tag = await FlutterNfcKit.poll(
      androidReaderModeFlags: flags,
      iosAlertMessage: 'แตะการ์ด NFC หรือแท็กเพื่ออ่านข้อมูล',
    );

    print('Tag ID: ${tag.id}');
    print('Tag Type: ${tag.type}');
    print('Standard: ${tag.standard}');

    // อ่าน NDEF Records (ถ้ามี)
    if (tag.ndefAvailable == true) {
      final ndefRecords = await FlutterNfcKit.readNDEFRecords(cached: false);
      for (final record in ndefRecords) {
        print('NDEF Record: ${record.payload}');
      }
    }
  } catch (e) {
    print('Error: $e');
  } finally {
    // สำคัญ: ต้องเรียก finish() เสมอ ไม่ว่าจะสำเร็จหรือไม่
    await FlutterNfcKit.finish(iosAlertMessage: 'อ่านข้อมูลสำเร็จ');
  }
}
```

---

### ตัวอย่างที่ 2: Android Performance Flags

```dart
// ลดความหน่วงในการอ่านแท็ก NFC บน Android ~500ms
// และปิดเสียง "beep" ของ platform
const int FLAG_READER_SKIP_NDEF_CHECK = 0x80;
const int FLAG_READER_NO_PLATFORM_SOUNDS = 0x100;
const int androidFlags = FLAG_READER_SKIP_NDEF_CHECK | FLAG_READER_NO_PLATFORM_SOUNDS;

final tag = await FlutterNfcKit.poll(
  androidReaderModeFlags: androidFlags,
);
```

---

### ตัวอย่างที่ 3: เขียน NDEF Record

```dart
import 'package:ndef/ndef.dart' as ndef;

Future<void> writeNdefRecord(String text) async {
  final tag = await FlutterNfcKit.poll(
    androidReaderModeFlags: 0x80 | 0x100,
    iosAlertMessage: 'แตะการ์ด NFC เพื่อเขียนข้อมูล',
  );

  if (tag.ndefWritable == true) {
    await FlutterNfcKit.writeNDEFRecords([
      ndef.TextRecord(text: text, language: 'th'),
    ]);
    print('เขียนข้อมูลสำเร็จ');
  } else {
    print('แท็กนี้ไม่รองรับการเขียน NDEF');
  }

  await FlutterNfcKit.finish(iosAlertMessage: 'เขียนข้อมูลสำเร็จ');
}
```

---

### ตัวอย่างที่ 4: ส่งคำสั่ง APDU (ISO 7816)

```dart
// ใช้สำหรับ Smart Card เช่น บัตรเครดิต EMV, บัตรประชาชน
Future<void> sendApduCommand() async {
  final tag = await FlutterNfcKit.poll(
    androidReaderModeFlags: 0x80 | 0x100,
    iosAlertMessage: 'แตะการ์ดสมาร์ทการ์ด',
  );

  // SELECT Application (EMV example)
  final response = await FlutterNfcKit.transceive(
    '00A4040007A0000000041010', // APDU command เป็น hex string
  );
  print('APDU Response: $response');

  await FlutterNfcKit.finish();
}
```

---

### ตัวอย่างที่ 5: อ่าน MIFARE Classic (Block Read)

```dart
Future<void> readMifareClassicBlock(int blockNumber) async {
  final tag = await FlutterNfcKit.poll(
    androidReaderModeFlags: 0x80 | 0x100,
  );

  if (tag.type == NFCTagType.mifare_classic) {
    // Authenticate sector ก่อน (Key A หรือ Key B)
    // แล้วค่อยอ่าน block
    final blockData = await FlutterNfcKit.transceive(
      '30${blockNumber.toRadixString(16).padLeft(2, '0')}', // READ command
    );
    print('Block $blockNumber: $blockData');
  }

  await FlutterNfcKit.finish();
}
```

---

## API Reference สำคัญ

### `FlutterNfcKit.poll()`

| Parameter | Type | Default | คำอธิบาย |
|---|---|---|---|
| `timeout` | `Duration` | 20 วินาที | หมดเวลารอแท็ก |
| `iosMultipleTagMessage` | `String?` | null | ข้อความเมื่อมีหลายแท็ก (iOS) |
| `iosAlertMessage` | `String?` | null | ข้อความบน iOS NFC sheet |
| `androidPollTimeout` | `Duration` | null | หมดเวลาเฉพาะ Android |
| `androidReaderModeFlags` | `int?` | null | Flags สำหรับ Android Reader Mode |

### `FlutterNfcKit.finish()`

| Parameter | Type | คำอธิบาย |
|---|---|---|
| `iosAlertMessage` | `String?` | ข้อความสำเร็จ (iOS) |
| `iosErrorMessage` | `String?` | ข้อความผิดพลาด (iOS) |

### `NFCTag` Properties

| Property | Type | คำอธิบาย |
|---|---|---|
| `id` | `String` | UID ของแท็ก (hex) |
| `type` | `NFCTagType` | ประเภทแท็ก |
| `standard` | `String` | มาตรฐาน ISO ที่ใช้ |
| `ndefAvailable` | `bool?` | มี NDEF data หรือไม่ |
| `ndefWritable` | `bool?` | เขียน NDEF ได้หรือไม่ |
| `ndefCapacity` | `int?` | ความจุ NDEF (bytes) |

---

## Best Practices

### Android
```dart
// ✅ ควรทำ: ใช้ androidReaderModeFlags เสมอเพื่อ performance
await FlutterNfcKit.poll(androidReaderModeFlags: 0x80 | 0x100);

// ❌ ไม่แนะนำ: ไม่ใส่ flags (ช้ากว่า ~500ms + มีเสียง beep)
await FlutterNfcKit.poll();
```

### iOS
```dart
// ✅ ควรทำ: ตรวจสอบ Info.plist ก่อน build ทุกครั้ง
// - NFCReaderUsageDescription
// - com.apple.developer.nfc.readersession.felica.systemcodes
// - com.apple.developer.nfc.readersession.iso7816.select-identifiers

// ✅ ควรทำ: ใส่ iosAlertMessage ให้ user รู้ว่าต้องทำอะไร
await FlutterNfcKit.poll(iosAlertMessage: 'แตะการ์ด NFC');

// ✅ ควรทำ: เรียก finish() เสมอใน finally block
try {
  final tag = await FlutterNfcKit.poll(...);
  // ...
} finally {
  await FlutterNfcKit.finish();
}
```

### Web (WebUSB)
```dart
// ⚠️ ระวัง: WebUSB ≠ WebNFC
// - ต้องมี USB NFC Reader เชื่อมต่ออยู่
// - รองรับเฉพาะ Chrome/Edge
// - ผู้ใช้ต้องอนุญาต USB device access ก่อน
```

### Error Handling
```dart
try {
  final tag = await FlutterNfcKit.poll(timeout: const Duration(seconds: 10));
  // ...
} on PlatformException catch (e) {
  if (e.code == '408') {
    print('หมดเวลา — ไม่พบแท็ก NFC');
  } else if (e.code == '400') {
    print('ยกเลิกโดยผู้ใช้');
  } else {
    print('NFC Error: ${e.code} — ${e.message}');
  }
} finally {
  await FlutterNfcKit.finish(iosErrorMessage: 'เกิดข้อผิดพลาด');
}
```

---

## Error Codes ที่พบบ่อย

| Code | ความหมาย | วิธีแก้ |
|---|---|---|
| `408` | Timeout — ไม่พบแท็กในเวลาที่กำหนด | เพิ่ม timeout หรือแจ้ง user ให้แตะช้าลง |
| `400` | ผู้ใช้ยกเลิก (กด Cancel บน iOS) | แจ้ง user ให้ลองอีกครั้ง |
| `500` | NFC ไม่พร้อม / ถูกปิดใช้งาน | ตรวจสอบ NFC settings ในอุปกรณ์ |
| `406` | แท็กไม่รองรับ operation นั้น | ตรวจสอบประเภทแท็กก่อนทำ operation |

---

## การ Debug

### Android
```bash
# ดู NFC logs
adb logcat | grep -i nfc

# ตรวจสอบว่าอุปกรณ์รองรับ NFC
adb shell pm list features | grep nfc
```

### iOS
```bash
# ดู CoreNFC logs ใน Xcode Console
# Filter: "CoreNFC" หรือ "NFCTagReaderSession"
```

### Flutter
```dart
// เปิด verbose logging
FlutterNfcKit.setVerboseLogging(true); // ถ้า plugin รองรับ

// หรือ wrap ด้วย try-catch แล้ว print error
print('Tag details: ${tag.toString()}');
```

---

## Changelog v3.6.2

- รองรับ Android Gradle Plugin 8.7
- รองรับ Java 17
- รองรับ Gradle 8.9
- แก้ไข bug บน iOS เมื่อ scan แท็กหลายครั้งติดกัน
- ปรับปรุง WebUSB stability

---

## ทรัพยากรเพิ่มเติม

- [pub.dev: flutter_nfc_kit](https://pub.dev/packages/flutter_nfc_kit)
- [GitHub: flutter_nfc_kit](https://github.com/nfcim/flutter_nfc_kit)
- [pub.dev: ndef](https://pub.dev/packages/ndef)
- [Android NFC Developer Guide](https://developer.android.com/guide/topics/connectivity/nfc)
- [Apple CoreNFC Documentation](https://developer.apple.com/documentation/corenfc)
