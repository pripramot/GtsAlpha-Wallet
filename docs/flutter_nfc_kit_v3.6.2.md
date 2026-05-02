# flutter_nfc_kit v3.6.2 — Knowledge Base (ภาษาไทย)

## ภาพรวม (Overview)

- Plugin สำหรับ Flutter รองรับ **Android**, **iOS** และ **Web** (ผ่าน WebUSB)
- ใช้สำหรับอ่าน/เขียน NFC Tags และ Smart Cards
- ใช้ `ndef` package ในการเข้ารหัส/ถอดรหัสข้อมูล NDEF

## ความสามารถหลัก (Features)

1. อ่าน/เขียน NDEF Records (ISO 14443 Type A/B, ISO 18092 FeliCa, ISO 15693)
2. อ่าน/เขียนระดับ Block/Page/Sector (MIFARE Classic/Ultralight บน Android, ISO 15693 บน iOS)
3. ส่งคำสั่ง Raw Commands (APDU ISO 7816 หรือ Layer 3 commands)
4. รองรับสองโหมดการทำงาน: **Polling** (ค่าเริ่มต้น) และ **Event Streaming** (Android เท่านั้น)

## การติดตั้ง (Installation)

```yaml
# pubspec.yaml
dependencies:
  flutter_nfc_kit: ^3.6.2
  ndef: ^0.3.1
```

```bash
flutter pub get
```

## การตั้งค่า (Setup)

### Android

- **Minimum Requirements:** Java 17, Gradle 8.9, Android SDK 24+, AGP 8.7
- ต้องกำหนด `jvmTarget` ที่สอดคล้องกันใน `build.gradle` ของแอป

เพิ่มใน `android/app/src/main/AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.NFC" />
<uses-feature android:name="android.hardware.nfc" android:required="true" />
```

### iOS

- **Minimum:** iOS 13+
- รองรับ Swift Package Manager

**ขั้นตอนการตั้งค่า:**

1. เพิ่ม Entitlement ใน `ios/Runner/Runner.entitlements`:

```xml
<key>com.apple.developer.nfc.readersession.formats</key>
<array>
    <string>NDEF</string>
    <string>TAG</string>
</array>
```

2. เพิ่มใน `ios/Runner/Info.plist`:

```xml
<!-- คำอธิบายสิทธิ์การใช้งาน NFC -->
<key>NFCReaderUsageDescription</key>
<string>ต้องการสิทธิ์ NFC เพื่ออ่านข้อมูลจากแท็ก</string>

<!-- สำหรับ FeliCa (ISO 18092) -->
<key>com.apple.developer.nfc.readersession.felica.systemcodes</key>
<array>
    <string>8008</string>
</array>

<!-- สำหรับ ISO 7816 Smart Cards -->
<key>com.apple.developer.nfc.readersession.iso7816.select-identifiers</key>
<array>
    <string>A0000002471001</string>
</array>
```

> ⚠️ **คำเตือน (iOS 14.5 และต่ำกว่า):** ต้องเพิ่ม `felica.systemcodes` และ `iso7816.select-identifiers` ก่อนเรียก `poll()` ที่มี `readIso18092` หรือ `readIso15693` เปิดใช้งาน มิฉะนั้น **NFC จะใช้งานไม่ได้จนกว่าจะรีบูต** ([CoreNFC bug #23](https://github.com/nfcim/flutter_nfc_kit/issues/23))

3. เปิด `Runner.xcworkspace` ใน Xcode → Project Settings → Signing & Capabilities
4. เลือก Runner target → กด "+ Capability" → เลือก **Near Field Communication Tag Reading**

### Web

Web version ใช้ **WebUSB protocol** ไม่ใช่ NFC ในเบราว์เซอร์โดยตรง  
ใช้สำหรับสื่อสารกับอุปกรณ์แบบ dual-interface (NFC/USB) เช่น Smart Card readers  
ศึกษา protocol เพิ่มเติมได้ที่ [WebUSB.md](https://github.com/nfcim/flutter_nfc_kit/blob/master/WebUSB.md)

## การใช้งานพื้นฐาน (Basic Usage)

### ตรวจสอบความพร้อมของ NFC

```dart
import 'package:flutter_nfc_kit/flutter_nfc_kit.dart';

final availability = await FlutterNfcKit.nfcAvailability;
if (availability != NFCAvailability.available) {
  // NFC ไม่พร้อมใช้งาน
  print('NFC ไม่รองรับหรือปิดใช้งานอยู่: $availability');
  return;
}
```

`NFCAvailability` มีค่าได้แก่:
| ค่า | ความหมาย |
|-----|----------|
| `available` | NFC พร้อมใช้งาน |
| `disabled` | NFC ปิดอยู่ (Android: ผู้ใช้ต้องเปิดเอง) |
| `not_supported` | อุปกรณ์ไม่รองรับ NFC |

### Polling — อ่านแท็ก NFC

```dart
import 'package:flutter_nfc_kit/flutter_nfc_kit.dart';
import 'package:ndef/ndef.dart' as ndef;
import 'dart:convert';

Future<void> readNfcTag() async {
  try {
    // เริ่ม session และรอแท็ก
    // timeout ใช้งานได้บน Android เท่านั้น
    final tag = await FlutterNfcKit.poll(
      timeout: const Duration(seconds: 10),
      iosMultipleTagMessage: "พบหลายแท็ก กรุณาแตะทีละแท็ก",
      iosAlertMessage: "แตะแท็ก NFC เพื่ออ่านข้อมูล",
    );

    print(jsonEncode(tag)); // แสดงข้อมูล tag ทั้งหมด
    print('Tag ID: ${tag.id}');
    print('Tag Type: ${tag.type}');

    // ตั้งข้อความบน iOS ระหว่าง session
    await FlutterNfcKit.setIosAlertMessage("กำลังอ่านข้อมูล...");

    // ... ดำเนินการกับ tag

  } catch (e) {
    print('Error: $e');
  } finally {
    // สิ้นสุด session เสมอ (เรียกครั้งเดียว)
    await FlutterNfcKit.finish(iosAlertMessage: "สำเร็จ");
  }
}
```

### NFCTag — ข้อมูลที่ได้จาก poll()

| Property | Type | คำอธิบาย |
|----------|------|----------|
| `id` | `String` | Tag UID (hex string) |
| `type` | `NFCTagType` | ประเภทแท็ก |
| `standard` | `String` | มาตรฐาน NFC |
| `ndefAvailable` | `bool` | มี NDEF records หรือไม่ |
| `ndefWritable` | `bool` | เขียน NDEF ได้หรือไม่ |
| `ndefCapacity` | `int?` | ขนาดสูงสุดของ NDEF (bytes) |
| `ndefType` | `String?` | ประเภทของ NDEF container |
| `mifareInfo` | `MifareInfo?` | ข้อมูล MIFARE (ถ้ามี) |
| `atqa` | `String?` | ATQA (ISO 14443 Type A) |
| `sak` | `int?` | SAK (ISO 14443 Type A) |
| `historicalBytes` | `String?` | Historical bytes (ISO 14443 Type A/B) |
| `protocolInfo` | `String?` | Protocol info (ISO 14443 Type B) |
| `applicationData` | `String?` | Application data (ISO 14443 Type B) |
| `hiLayerResponse` | `String?` | Higher layer response (ISO 14443 Type B) |
| `systemCode` | `String?` | System code (FeliCa) |
| `dsfId` | `int?` | DSFID (ISO 15693) |
| `icReference` | `int?` | IC Reference (ISO 15693) |

### NFCTagType — ประเภทแท็กที่รองรับ

| ค่า | มาตรฐาน |
|-----|---------|
| `iso7816` | ISO 7816 Smart Cards (APDU) |
| `iso15693` | ISO 15693 (NFC-V) |
| `iso18092` | ISO 18092 / FeliCa (NFC-F) |
| `mifare_classic` | MIFARE Classic |
| `mifare_ultralight` | MIFARE Ultralight |
| `mifare_desfire` | MIFARE DESFire |
| `unknown` | ไม่ทราบประเภท |

## การทำงานกับ NDEF (NDEF Operations)

### อ่าน NDEF Records

```dart
if (tag.ndefAvailable) {
  // อ่านแบบ decoded (ให้ NDEFRecord objects)
  // ตัวอย่าง: UriRecord: id=(empty) typeNameFormat=TypeNameFormat.nfcWellKnown type=U uri=https://example.com
  final records = await FlutterNfcKit.readNDEFRecords(cached: false);
  for (final record in records) {
    print(record.toString());
  }

  // อ่านแบบ raw (ข้อมูล hex string)
  // ตัวอย่าง: {"identifier":"","payload":"00010203","type":"0001","typeNameFormat":"nfcWellKnown"}
  final rawRecords = await FlutterNfcKit.readNDEFRawRecords(cached: false);
  for (final record in rawRecords) {
    print(jsonEncode(record));
  }
}
```

> **`cached: false`** — บังคับอ่านใหม่จากแท็ก (ไม่ใช้ cache จาก poll)  
> **`cached: true`** — ใช้ข้อมูลที่อ่านไว้ตอน poll แล้ว (เร็วกว่า)

### เขียน NDEF Records

```dart
import 'package:ndef/ndef.dart' as ndef;

if (tag.ndefWritable) {
  // เขียน URI record
  await FlutterNfcKit.writeNDEFRecords([
    ndef.UriRecord.fromUriString("https://github.com/nfcim/flutter_nfc_kit"),
  ]);

  // เขียน Text record
  await FlutterNfcKit.writeNDEFRecords([
    ndef.TextRecord(text: "สวัสดี GtsAlpha!", language: "th"),
  ]);

  // เขียน raw NDEF records
  await FlutterNfcKit.writeNDEFRawRecords([
    NDEFRawRecord("00", "0001", "0002", "0003", ndef.TypeNameFormat.unknown),
  ]);
}
```

### NDEF Record Types ที่ใช้บ่อย (จาก `ndef` package)

| Record Type | คลาส | ใช้สำหรับ |
|-------------|------|----------|
| URI | `ndef.UriRecord` | เก็บ URL หรือ URI |
| Text | `ndef.TextRecord` | เก็บข้อความพร้อม language code |
| MIME | `ndef.MimeRecord` | เก็บข้อมูล MIME type ใดก็ได้ |
| External Type | `ndef.ExternalRecord` | Custom type ของผู้พัฒนา |
| Smart Poster | `ndef.SmartPosterRecord` | รวม URI + Title + Action |

## การทำงานกับ MIFARE (MIFARE Operations) — Android

### MIFARE Classic

```dart
if (tag.type == NFCTagType.mifare_classic) {
  // 1. ยืนยันตัวตน sector ก่อนอ่าน/เขียน
  // keyA ค่าเริ่มต้น: FFFFFFFFFFFF
  await FlutterNfcKit.authenticateSector(0, keyA: "FFFFFFFFFFFF");

  // 2. อ่านข้อมูล
  final sectorData = await FlutterNfcKit.readSector(0);  // อ่าน 1 sector
  final blockData  = await FlutterNfcKit.readBlock(0);   // อ่าน 1 block

  // 3. เขียนข้อมูล
  await FlutterNfcKit.writeBlock(1, [0xDE, 0xAD, 0xBE, 0xEF, ...]);
}
```

### MIFARE Ultralight / ISO 15693

```dart
// ISO 15693 (NFC-V) — iOS เท่านั้น
if (tag.type == NFCTagType.iso15693) {
  // เขียน block
  await FlutterNfcKit.writeBlock(
    1,                              // block index
    [0xDE, 0xAD, 0xBE, 0xFF],      // data (4 bytes)
    iso15693RequestFlag: Iso15693RequestFlag(),  // optional flags
    iso15693ExtendedMode: false,    // ใช้ extended mode หรือไม่
  );

  // อ่าน block
  final data = await FlutterNfcKit.readBlock(1);
}
```

## การส่งคำสั่ง Raw (Raw Commands / Transceive)

ใช้สำหรับส่งคำสั่ง APDU (ISO 7816) หรือ Layer 3 commands ตรงๆ กับแท็ก

```dart
// ISO 7816 APDU — อ่านข้อมูล (READ BINARY)
// timeout ใช้งานได้บน Android เท่านั้น
final response = await FlutterNfcKit.transceive(
  "00B0950000",                    // APDU hex string
  const Duration(seconds: 5),      // timeout (Android only)
);
print(response); // hex string ของ response

// ตัวอย่าง: SELECT AID
final selectResponse = await FlutterNfcKit.transceive("00A4040007A0000000031010");
print(selectResponse); // เช่น "9000" = success
```

### Error Codes ที่พบบ่อย

Error codes มีความหมายคล้าย HTTP Status Codes:

| Code | ความหมาย |
|------|----------|
| `400` | Bad request / Invalid parameters |
| `404` | Tag not found / Session expired |
| `405` | Method not supported on this tag/platform |
| `408` | Timeout (Android only) |
| `500` | Internal / hardware error |
| `503` | NFC disabled หรือ not available |

ข้อผิดพลาดจะถูก throw เป็น `PlatformException` พร้อม `code` และ `message`

## Event Streaming Mode — Android เท่านั้น

โหมดนี้ช่วยให้รับ NFC tag events ต่อเนื่องในขณะที่แอปทำงานอยู่เบื้องหน้า

### ขั้นตอนที่ 1: สร้าง Custom Activity (Kotlin)

สร้างไฟล์ `android/app/src/main/kotlin/your/package/name/MainActivity.kt`:

```kotlin
package your.package.name

import android.app.PendingIntent
import android.content.Intent
import android.nfc.NfcAdapter
import android.nfc.Tag
import io.flutter.embedding.android.FlutterActivity
import im.nfc.flutter_nfc_kit.FlutterNfcKitPlugin

class MainActivity : FlutterActivity() {
    override fun onResume() {
        super.onResume()
        val adapter: NfcAdapter? = NfcAdapter.getDefaultAdapter(this)
        val pendingIntent: PendingIntent = PendingIntent.getActivity(
            this, 0,
            Intent(this, javaClass).addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP),
            PendingIntent.FLAG_MUTABLE
        )
        adapter?.enableForegroundDispatch(this, pendingIntent, null, null)
    }

    override fun onPause() {
        super.onPause()
        val adapter: NfcAdapter? = NfcAdapter.getDefaultAdapter(this)
        adapter?.disableForegroundDispatch(this)
    }

    override fun onNewIntent(intent: Intent) {
        val tag: Tag? = intent.getParcelableExtra(NfcAdapter.EXTRA_TAG)
        tag?.apply(FlutterNfcKitPlugin::handleTag)
    }
}
```

### ขั้นตอนที่ 2: อัปเดต AndroidManifest.xml

```xml
<activity
    android:name=".MainActivity"
    ...>
```

### ขั้นตอนที่ 3: ฟัง Stream ใน Flutter

```dart
@override
void initState() {
  super.initState();
  FlutterNfcKit.tagStream.listen((tag) {
    print('พบแท็ก: ${tag.id}');
    // ทำงานกับ tag ได้เลย
    FlutterNfcKit.transceive("xxx");
    // ⚠️ ห้ามเรียก FlutterNfcKit.finish() ในโหมดนี้
  });
}
```

## การปรับแต่งประสิทธิภาพ (Performance Optimization) — Android

ใช้ `androidReaderModeFlags` เพื่อเพิ่มความเร็วในการตรวจจับแท็ก

```dart
// ข้ามการตรวจสอบ NDEF อัตโนมัติ (~500ms เร็วขึ้น)
// และปิดเสียง/การสั่น default ของระบบ
const flags = 0x80 | 0x100;
// 0x80  = FLAG_READER_SKIP_NDEF_CHECK
// 0x100 = FLAG_READER_NO_PLATFORM_SOUNDS

final tag = await FlutterNfcKit.poll(
  androidReaderModeFlags: flags,
);
```

| Flag | ค่า hex | คำอธิบาย |
|------|---------|----------|
| `FLAG_READER_SKIP_NDEF_CHECK` | `0x80` | ข้าม NDEF auto-discovery เร็วขึ้น ~500ms |
| `FLAG_READER_NO_PLATFORM_SOUNDS` | `0x100` | ปิดเสียงบี๊บ/การสั่นของระบบ |
| `FLAG_READER_NFC_A` | `0x01` | เปิดใช้ ISO 14443 Type A |
| `FLAG_READER_NFC_B` | `0x02` | เปิดใช้ ISO 14443 Type B |
| `FLAG_READER_NFC_F` | `0x04` | เปิดใช้ ISO 18092 / FeliCa |
| `FLAG_READER_NFC_V` | `0x08` | เปิดใช้ ISO 15693 |

## สรุป API หลัก (API Quick Reference)

| Method | พารามิเตอร์หลัก | คืนค่า | หมายเหตุ |
|--------|---------------|--------|---------|
| `FlutterNfcKit.nfcAvailability` | — | `NFCAvailability` | ตรวจสอบสถานะ NFC |
| `FlutterNfcKit.poll(...)` | `timeout`, `iosAlertMessage`, `androidReaderModeFlags` | `NFCTag` | เริ่ม session รอแท็ก |
| `FlutterNfcKit.finish(...)` | `iosAlertMessage`, `iosErrorMessage` | `void` | สิ้นสุด session |
| `FlutterNfcKit.setIosAlertMessage(msg)` | `String` | `void` | เปลี่ยนข้อความ iOS ระหว่าง session |
| `FlutterNfcKit.readNDEFRecords(...)` | `cached` | `List<ndef.NDEFRecord>` | อ่าน NDEF decoded |
| `FlutterNfcKit.readNDEFRawRecords(...)` | `cached` | `List<NDEFRawRecord>` | อ่าน NDEF raw hex |
| `FlutterNfcKit.writeNDEFRecords(records)` | `List<ndef.NDEFRecord>` | `void` | เขียน NDEF decoded |
| `FlutterNfcKit.writeNDEFRawRecords(records)` | `List<NDEFRawRecord>` | `void` | เขียน NDEF raw |
| `FlutterNfcKit.transceive(data, ...)` | `String` (hex), `timeout` | `String` (hex) | ส่ง raw command |
| `FlutterNfcKit.authenticateSector(...)` | `index`, `keyA`/`keyB` | `void` | MIFARE authenticate |
| `FlutterNfcKit.readBlock(index)` | `int` | `Uint8List` | อ่าน 1 block |
| `FlutterNfcKit.readSector(index)` | `int` | `Uint8List` | อ่าน 1 sector (MIFARE) |
| `FlutterNfcKit.writeBlock(index, data, ...)` | `int`, `List<int>` | `void` | เขียน 1 block |
| `FlutterNfcKit.tagStream` | — | `Stream<NFCTag>` | stream แท็ก (Android) |

## การรวมเข้ากับ GtsAlpha Wallet

ตัวอย่างการใช้ flutter_nfc_kit แทน nfc_manager ในโปรเจกต์นี้:

```dart
// lib/services/nfc_service.dart
import 'package:flutter_nfc_kit/flutter_nfc_kit.dart';
import 'package:ndef/ndef.dart' as ndef;

class NfcService {
  /// ตรวจสอบว่าอุปกรณ์รองรับ NFC หรือไม่
  Future<bool> isAvailable() async {
    final availability = await FlutterNfcKit.nfcAvailability;
    return availability == NFCAvailability.available;
  }

  /// อ่าน NFC Tag แล้ว return ข้อมูล NDEF เป็น String
  Future<String?> readNfcTag() async {
    try {
      final tag = await FlutterNfcKit.poll(
        timeout: const Duration(seconds: 10),
        iosAlertMessage: "แตะการ์ดหรือแหวน NFC",
      );

      String? result;

      if (tag.ndefAvailable) {
        final records = await FlutterNfcKit.readNDEFRecords(cached: false);
        if (records.isNotEmpty) {
          result = records.first.toString();
        }
      }

      await FlutterNfcKit.finish(iosAlertMessage: "อ่านข้อมูลสำเร็จ");
      return result;
    } catch (e) {
      await FlutterNfcKit.finish(iosErrorMessage: "เกิดข้อผิดพลาด");
      rethrow;
    }
  }

  /// เขียน URI ลงบน NFC Tag
  Future<void> writeUri(String uri) async {
    try {
      final tag = await FlutterNfcKit.poll(
        iosAlertMessage: "แตะแท็ก NFC เพื่อเขียนข้อมูล",
      );

      if (!tag.ndefWritable) {
        throw Exception("แท็กนี้ไม่รองรับการเขียน NDEF");
      }

      await FlutterNfcKit.writeNDEFRecords([
        ndef.UriRecord.fromUriString(uri),
      ]);

      await FlutterNfcKit.finish(iosAlertMessage: "เขียนข้อมูลสำเร็จ");
    } catch (e) {
      await FlutterNfcKit.finish(iosErrorMessage: "เกิดข้อผิดพลาด");
      rethrow;
    }
  }
}
```

## การแก้ไขปัญหา (Troubleshooting)

### Android

| ปัญหา | วิธีแก้ |
|-------|---------|
| Build ล้มเหลว: Java version | ตรวจสอบ `jvmTarget` ใน `build.gradle` ให้ตรงกับ Java 17 |
| แท็กไม่ถูกตรวจจับ | ตรวจสอบว่าเพิ่ม `NFC permission` ใน `AndroidManifest.xml` แล้ว |
| ตรวจจับช้า | ใช้ `androidReaderModeFlags: 0x80` เพื่อข้าม NDEF check |
| Crash บน `finish()` | ตรวจสอบว่าเรียก `finish()` เพียงครั้งเดียวต่อ session |

### iOS

| ปัญหา | วิธีแก้ |
|-------|---------|
| NFC ใช้งานไม่ได้หลัง poll | เพิ่ม `felica.systemcodes` และ `iso7816.select-identifiers` ใน Info.plist ก่อน (iOS 14.5-) |
| ไม่เห็น NFC capability | ใน Xcode: Signing & Capabilities → "+ Capability" → Near Field Communication Tag Reading |
| `NFCReaderUsageDescription` missing | เพิ่ม key นี้ใน `Info.plist` |

### ทั่วไป

| ปัญหา | วิธีแก้ |
|-------|---------|
| `PlatformException code: 404` | Session หมดเวลา หรือแท็กออกจากระยะ NFC ก่อนอ่านเสร็จ |
| `PlatformException code: 405` | ฟังก์ชันนี้ไม่รองรับบนแพลตฟอร์ม/แท็กนี้ |
| NDEF เขียนไม่ได้ | ตรวจสอบ `tag.ndefWritable` ก่อนเขียน บางแท็กล็อกการเขียนไว้ |

## ลิงก์อ้างอิง (References)

- [pub.dev — flutter_nfc_kit](https://pub.dev/packages/flutter_nfc_kit)
- [GitHub — nfcim/flutter_nfc_kit](https://github.com/nfcim/flutter_nfc_kit)
- [Example Code](https://github.com/nfcim/flutter_nfc_kit/blob/master/example/example.md)
- [pub.dev — ndef package](https://pub.dev/packages/ndef)
- [WebUSB Protocol](https://github.com/nfcim/flutter_nfc_kit/blob/master/WebUSB.md)
- [Android NFC Developer Guide](https://developer.android.com/guide/topics/connectivity/nfc)
- [Apple CoreNFC Documentation](https://developer.apple.com/documentation/corenfc)
