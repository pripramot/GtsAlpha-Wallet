# คู่มือเริ่มต้นใช้งาน NFC ด้วย flutter_nfc_kit (ภาษาไทย)

> เวอร์ชัน: `flutter_nfc_kit ^3.6.2` | Flutter `>=3.3.0`

---

## 1. ติดตั้ง

```bash
flutter pub add flutter_nfc_kit
flutter pub add ndef
```

หรือเพิ่มใน `pubspec.yaml` โดยตรง:

```yaml
dependencies:
  flutter_nfc_kit: ^3.6.2
  ndef: ^0.3.1
```

แล้วรัน:

```bash
flutter pub get
```

---

## 2. ตั้งค่า Android

**`android/app/src/main/AndroidManifest.xml`** — เพิ่ม permission:

```xml
<manifest ...>
    <uses-permission android:name="android.permission.NFC" />
    <uses-feature android:name="android.hardware.nfc" android:required="true" />
    ...
</manifest>
```

**ข้อกำหนดขั้นต่ำ:**
- Android SDK 24+ (Android 7.0)
- Java 17
- Gradle 8.9 / Android Gradle Plugin 8.7

---

## 3. ตั้งค่า iOS

### ขั้นตอนที่ 1 — เปิด NFC Capability ใน Xcode
1. เปิดโปรเจกต์ใน Xcode
2. ไปที่ **Target → Signing & Capabilities**
3. กด **+ Capability** → เลือก **Near Field Communication Tag Reading**

### ขั้นตอนที่ 2 — เพิ่ม Info.plist keys

**`ios/Runner/Info.plist`:**

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

> ⚠️ **สำคัญมาก:** ถ้าไม่ใส่ key เหล่านี้ก่อนเรียก `poll()` iOS จะ **ค้างจนต้อง reboot** — นี่คือ bug ของ iOS CoreNFC

---

## 4. ตัวอย่างโค้ดอ่าน NDEF

```dart
import 'package:flutter_nfc_kit/flutter_nfc_kit.dart';
import 'package:ndef/ndef.dart' as ndef;

Future<void> readNfcTag() async {
  // ตรวจสอบ NFC availability
  final availability = await FlutterNfcKit.nfcAvailability;
  if (availability != NFCAvailability.available) {
    print('NFC ไม่พร้อม: $availability');
    return;
  }

  try {
    // Poll หาแท็ก (Android: ใช้ flags เพื่อ performance + ปิดเสียง)
    final tag = await FlutterNfcKit.poll(
      androidReaderModeFlags: 0x80 | 0x100,
      iosAlertMessage: 'แตะการ์ด NFC ของคุณ',
    );

    print('พบแท็ก: ${tag.id} (${tag.type})');

    // อ่าน NDEF ถ้ามี
    if (tag.ndefAvailable == true) {
      final records = await FlutterNfcKit.readNDEFRecords(cached: false);
      for (final record in records) {
        if (record is ndef.TextRecord) {
          print('ข้อความ: ${record.text}');
        } else if (record is ndef.UriRecord) {
          print('URL: ${record.uri}');
        }
      }
    } else {
      print('แท็กนี้ไม่มีข้อมูล NDEF');
    }

    await FlutterNfcKit.finish(iosAlertMessage: 'อ่านสำเร็จ ✓');
  } catch (e) {
    await FlutterNfcKit.finish(iosErrorMessage: 'เกิดข้อผิดพลาด');
    print('Error: $e');
  }
}
```

---

## 5. เขียนข้อมูล NDEF

```dart
Future<void> writeNfcTag(String text) async {
  try {
    final tag = await FlutterNfcKit.poll(
      androidReaderModeFlags: 0x80 | 0x100,
      iosAlertMessage: 'แตะการ์ดเพื่อเขียนข้อมูล',
    );

    if (tag.ndefWritable != true) {
      print('แท็กนี้ไม่สามารถเขียนได้');
      await FlutterNfcKit.finish(iosErrorMessage: 'ไม่สามารถเขียนได้');
      return;
    }

    await FlutterNfcKit.writeNDEFRecords([
      ndef.TextRecord(text: text, language: 'th'),
    ]);

    await FlutterNfcKit.finish(iosAlertMessage: 'เขียนสำเร็จ ✓');
  } catch (e) {
    await FlutterNfcKit.finish(iosErrorMessage: 'เขียนล้มเหลว');
    print('Error: $e');
  }
}
```

---

## 6. Debug ปัญหาที่พบบ่อย

| ปัญหา | สาเหตุ | วิธีแก้ |
|---|---|---|
| iOS ค้าง ต้อง reboot | Info.plist keys ขาด | เพิ่ม `felica.systemcodes` และ `iso7816.select-identifiers` |
| `PlatformException(408)` | หมดเวลา ไม่พบแท็ก | เพิ่ม `timeout` parameter หรือแตะให้ช้าลง |
| `PlatformException(400)` | ผู้ใช้กด Cancel | แจ้งให้ลองอีกครั้ง |
| Android ช้า / มีเสียง beep | ไม่ได้ใส่ `androidReaderModeFlags` | ใส่ `0x80` OR `0x100` (รวมเป็น `0x180`) |
| Web ไม่ทำงาน | WebUSB ต้องมี USB Reader | เชื่อมต่อ USB NFC Reader และใช้ Chrome/Edge |
| NFC ไม่ตอบสนอง Android | NFC ถูกปิด | ไปที่ Settings → Connections → NFC → เปิด |
| iOS NFC sheet ไม่ขึ้น | ไม่ได้เปิด NFC Capability | เปิดใน Xcode: Signing & Capabilities |

### ตรวจสอบ Android logs:
```bash
adb logcat | grep -i nfc
```

### ตรวจสอบ iOS logs:
- เปิด Xcode → Window → Devices and Simulators → ดู Console
- กรอง keyword: `CoreNFC` หรือ `NFCTagReaderSession`

---

## 7. NFC Availability States

```dart
final availability = await FlutterNfcKit.nfcAvailability;

switch (availability) {
  case NFCAvailability.available:
    print('NFC พร้อมใช้งาน');
    break;
  case NFCAvailability.disabled:
    print('NFC ถูกปิด — แนะนำให้ไปเปิดใน Settings');
    break;
  case NFCAvailability.not_supported:
    print('อุปกรณ์นี้ไม่รองรับ NFC');
    break;
}
```

---

## 8. Error Handling ครบถ้วน

```dart
try {
  final tag = await FlutterNfcKit.poll(
    timeout: const Duration(seconds: 15),
    androidReaderModeFlags: 0x80 | 0x100,
    iosAlertMessage: 'แตะการ์ด NFC',
  );
  // ทำงานกับ tag...
} on PlatformException catch (e) {
  switch (e.code) {
    case '408':
      // Timeout
      print('ไม่พบแท็ก NFC ภายในเวลาที่กำหนด');
      break;
    case '400':
      // User cancelled
      print('ยกเลิกการอ่าน');
      break;
    case '500':
      // NFC unavailable
      print('NFC ไม่พร้อมใช้งาน');
      break;
    default:
      print('เกิดข้อผิดพลาด: ${e.code} - ${e.message}');
  }
} finally {
  // เรียก finish() เสมอ แม้จะเกิด error
  await FlutterNfcKit.finish(iosErrorMessage: 'เสร็จสิ้น');
}
```

---

## 9. ทรัพยากรเพิ่มเติม

- 📖 [Knowledge Base: flutter_nfc_kit v3.6.2](./flutter_nfc_kit_v3.6.2.md)
- 🌐 [pub.dev: flutter_nfc_kit](https://pub.dev/packages/flutter_nfc_kit)
- 📦 [pub.dev: ndef](https://pub.dev/packages/ndef)
- 🐛 [GitHub Issues](https://github.com/nfcim/flutter_nfc_kit/issues)
