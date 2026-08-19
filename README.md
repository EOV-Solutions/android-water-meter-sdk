# 📱 Water Meter SDK cho Android Native

> SDK quét số đồng hồ nước sử dụng AI cho ứng dụng Android native (Java/Kotlin).

[![Nền tảng](https://img.shields.io/badge/platform-Android-green.svg)](https://www.android.com/)
[![Android](https://img.shields.io/badge/Android-%3E%3D6.0-brightgreen.svg)](https://www.android.com/)
[![Phiên bản](https://img.shields.io/badge/version-2.1-blue.svg)](repo/com/eov/water-meter-sdk/)
[![Giấy phép](https://img.shields.io/badge/license-EOV-orange.svg)](#-giấy-phép)

Repo này là **Maven repository phân phối** — thư mục [`repo/`](repo/) chứa file AAR build sẵn kèm POM. Không cần tải file thủ công, chỉ cần khai báo dependency trong Gradle.

## ✨ Tính năng

- 📷 Xem trước camera thời gian thực với lớp phủ AI
- 🎯 Tự động phát hiện với ngưỡng độ tin cậy
- ⚡ Tự động chụp khi căn chỉnh đúng
- 🔦 Hỗ trợ đèn flash
- 🔍 Điều khiển zoom
- 📐 Phát hiện OBB (hình chữ nhật bao quanh)
- 💾 Lưu ảnh và trả về đường dẫn
- 📦 Model AI đóng gói sẵn trong AAR — không cần copy model thủ công
- 🎨 Tùy chỉnh màn hình quét (tiêu đề, nút đóng, tự đóng khi có kết quả)
- 🔑 Quản lý license tích hợp

## 🛠️ Yêu cầu

- Android 6.0 trở lên (`minSdk` 23)
- ABI hỗ trợ: `armeabi-v7a`, `arm64-v8a`
- Android Gradle Plugin hỗ trợ Java 17 bytecode (AGP 8.x khuyến nghị)
- License key do EOV Solutions cấp

## 🚀 Cài đặt

### Bước 1: Thêm Maven repository

```gradle
// settings.gradle
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven { url 'https://raw.githubusercontent.com/EOV-Solutions/android-water-meter-sdk/main/repo' }
    }
}
```

### Bước 2: Thêm dependency

```gradle
// app/build.gradle
dependencies {
    implementation 'com.eov:water-meter-sdk:2.1'
}
```

Gradle sẽ tự tải AAR và các dependency đi kèm (`androidx.appcompat`, `androidx.constraintlayout`).

### Bước 3: Sync và build

```bash
./gradlew build
```

### Chỉ với bản 2.0: ghi đè theme trong AndroidManifest.xml

> Từ bản **2.1**, SDK không còn khai `android:theme` ở tag `<application>` — bỏ qua bước này. Bước này chỉ cần khi bạn buộc phải dùng bản 2.0.

Bản 2.0 khai `android:theme` ở tag `<application>` của AAR, nên nếu app của bạn cũng khai theme riêng thì phải thêm `tools:replace` để tránh lỗi manifest merger:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <application
        android:theme="@style/Theme.AppCompat.Light.DarkActionBar"
        tools:replace="android:theme">
        ...
    </application>
</manifest>
```

### Kiểm tra cài đặt

Quyền (`CAMERA`, `INTERNET`...) và các Activity của SDK đã được khai báo sẵn trong AAR — manifest merger tự gộp vào app, **không cần khai báo thêm** trong `AndroidManifest.xml`. Kiểm tra bằng:

```bash
./gradlew :app:dependencies --configuration releaseRuntimeClasspath | grep water-meter
# Phải thấy: com.eov:water-meter-sdk:2.0
```

### Nếu app bật minify (Proguard/R8)

```proguard
-keep class com.eov.watermeter.** { *; }
-keep class com.baidu.paddle.lite.** { *; }
```

### Nếu app có native library riêng

```gradle
// app/build.gradle
android {
    packagingOptions {
        pickFirst '**/libc++_shared.so'
    }
}
```

## 💻 Sử dụng

### ⚠️ Khởi tạo License (Bắt buộc)

**QUAN TRỌNG**: Bạn phải khởi tạo license trước khi sử dụng tính năng quét. Nếu license chưa kích hoạt, camera vẫn mở nhưng OCR sẽ không chạy.

```java
import com.eov.watermeter.WaterMeterSDK;

// Khởi tạo cơ bản (gọi khi app start, ví dụ trong onCreate của Application/Activity)
WaterMeterSDK.initialize(context, "YOUR_LICENSE_KEY",
    new WaterMeterSDK.LicenseCallback() {
        @Override
        public void onSuccess() {
            Log.d("SDK", "✓ License activated");
            // SDK sẵn sàng sử dụng
        }

        @Override
        public void onError(String error) {
            Log.e("SDK", "✗ License error: " + error);
        }
    });
```

**HOẶC** với metadata, device user và mã tổ chức (để theo dõi trên admin) — cả 3 đều optional, có thể truyền 1 trong số đó, tất cả, hoặc không truyền:

```java
JSONObject metadata = new JSONObject();
metadata.put("location", "Chi nhánh Quận 1");
metadata.put("customerId", "12345");

WaterMeterSDK.initialize(
    context,
    "YOUR_LICENSE_KEY",
    metadata,                   // metadataInfo (optional, có thể null)
    "nhanvien@congty.com",      // deviceUser (optional, có thể null)
    "MA_TO_CHUC_123",           // maToChuc (optional, có thể null)
    callback
);
```

Kiểm tra license hiện tại:

```java
boolean valid = WaterMeterSDK.isLicenseValid();
int status = WaterMeterSDK.getLicenseStatus();
String message = WaterMeterSDK.getStatusMessage();
// Status codes:
// 0 = not initialized
// 1 = valid
// 2 = expired
// 3 = grace period
// 4 = invalid
// 5 = blocked
// 6 = quota exceeded
```

### Ví dụ cơ bản

```java
public class MainActivity extends AppCompatActivity {

    private static final int REQUEST_SCAN = 1001;

    private void startScan() {
        WaterMeterSDK.startCameraScan(this, REQUEST_SCAN);
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);

        if (requestCode == REQUEST_SCAN && resultCode == RESULT_OK && data != null) {
            String text = data.getStringExtra(WaterMeterSDK.EXTRA_RESULT_TEXT);
            float confidence = data.getFloatExtra(WaterMeterSDK.EXTRA_RESULT_CONFIDENCE, 0f);
            String imagePath = data.getStringExtra(WaterMeterSDK.EXTRA_RESULT_IMAGE_PATH);

            Log.d("Scan", "✓ Số: " + text);
            Log.d("Scan", "Độ tin cậy: " + (confidence * 100) + "%");
            Log.d("Scan", "Ảnh lưu tại: " + imagePath);
        } else if (requestCode == REQUEST_SCAN) {
            Log.d("Scan", "Người dùng đã hủy");
        }
    }
}
```

### Ví dụ nâng cao với tuỳ chọn

```java
WaterMeterSDK.startCameraScan(this, REQUEST_SCAN,
    new WaterMeterSDK.CameraScanBuilder()
        .setTitle("Quét số đồng hồ nước")   // Tiêu đề màn hình quét
        .setShowCloseButton(true)            // Hiện nút đóng (X)
        .setAutoCloseOnResult(true)          // Tự động đóng khi quét thành công
        .setImageMaxWidth(1920)              // Resize ảnh về max width (giữ tỷ lệ)
        .setImageMaxHeight(1080)             // Resize ảnh về max height (giữ tỷ lệ)
);
```

### Tích hợp hoàn chỉnh

```java
import android.content.Intent;
import android.os.Bundle;
import android.util.Log;
import android.widget.Button;
import android.widget.ImageView;
import android.widget.TextView;
import android.widget.Toast;

import androidx.appcompat.app.AppCompatActivity;

import com.eov.watermeter.WaterMeterSDK;

import org.json.JSONObject;

public class MainActivity extends AppCompatActivity {

    private static final int REQUEST_SCAN = 1001;

    private TextView meterValue;
    private TextView confidenceValue;
    private ImageView capturedImage;
    private boolean sdkReady = false;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        meterValue = findViewById(R.id.meter_value);
        confidenceValue = findViewById(R.id.confidence);
        capturedImage = findViewById(R.id.captured_image);

        Button scanButton = findViewById(R.id.btn_scan);
        scanButton.setOnClickListener(v -> {
            if (sdkReady) {
                WaterMeterSDK.startCameraScan(this, REQUEST_SCAN,
                    new WaterMeterSDK.CameraScanBuilder()
                        .setTitle("Quét số đồng hồ nước")
                        .setAutoCloseOnResult(true)
                        .setImageMaxWidth(1920));
            } else {
                Toast.makeText(this, "License chưa kích hoạt", Toast.LENGTH_SHORT).show();
            }
        });

        initializeSdk();
    }

    private void initializeSdk() {
        WaterMeterSDK.initialize(this, "YOUR_LICENSE_KEY",
            new WaterMeterSDK.LicenseCallback() {
                @Override
                public void onSuccess() {
                    sdkReady = true;
                    Log.d("SDK", "License activated");
                }

                @Override
                public void onError(String error) {
                    Log.e("SDK", "License error: " + error);
                    Toast.makeText(MainActivity.this,
                        "Không thể kích hoạt license: " + error, Toast.LENGTH_LONG).show();
                }
            });
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);

        if (requestCode == REQUEST_SCAN && resultCode == RESULT_OK && data != null) {
            String text = data.getStringExtra(WaterMeterSDK.EXTRA_RESULT_TEXT);
            float confidence = data.getFloatExtra(WaterMeterSDK.EXTRA_RESULT_CONFIDENCE, 0f);
            String imagePath = data.getStringExtra(WaterMeterSDK.EXTRA_RESULT_IMAGE_PATH);

            meterValue.setText(text);
            confidenceValue.setText(String.format("%.1f%%", confidence * 100));

            if (imagePath != null) {
                capturedImage.setImageURI(android.net.Uri.parse("file://" + imagePath));
            }
        }
    }
}
```

## 📖 API

### `WaterMeterSDK.initialize(context, licenseKey, callback)`

**⚠️ BẮT BUỘC** — Khởi tạo SDK với license key. Phải gọi trước khi sử dụng tính năng quét.

**Tham số:**
- `context` (Context) — Application context
- `licenseKey` (String, bắt buộc) — License key từ backend
- `callback` (LicenseCallback) — Callback kết quả:
  - `onSuccess()` — Kích hoạt thành công, SDK sẵn sàng
  - `onError(String errorMessage)` — Có lỗi (key sai, hết hạn, không có mạng lần đầu...)

**Các overload với tham số tùy chọn (cho admin tracking):**

```java
// + metadata và device user
WaterMeterSDK.initialize(context, licenseKey, metadataInfo, deviceUser, callback);

// + metadata, device user và mã tổ chức
WaterMeterSDK.initialize(context, licenseKey, metadataInfo, deviceUser, maToChuc, callback);
```

- `metadataInfo` (JSONObject, tùy chọn) — Metadata gửi lên server để admin theo dõi
- `deviceUser` (String, tùy chọn) — Email/ID người dùng thiết bị
- `maToChuc` (String, tùy chọn) — Mã tổ chức của thiết bị

Tham số nào không dùng thì truyền `null`.

---

### `WaterMeterSDK.isLicenseValid()`

Kiểm tra license hiện tại có hợp lệ không (bao gồm cả trường hợp bị khóa do vượt quota).

**Trả về:** `boolean` — `true` nếu license hợp lệ và không bị khóa.

---

### `WaterMeterSDK.getLicenseStatus()` / `getStatusMessage()`

Lấy mã trạng thái license và thông báo tương ứng.

```java
int status = WaterMeterSDK.getLicenseStatus();
String message = WaterMeterSDK.getStatusMessage();       // của trạng thái hiện tại
String message2 = WaterMeterSDK.getStatusMessage(status); // của một mã bất kỳ
```

**Status Codes:**

| Mã | Hằng số | Ý nghĩa |
|----|---------|---------|
| 0 | NOT_INITIALIZED | SDK chưa khởi tạo |
| 1 | VALID | License hợp lệ |
| 2 | EXPIRED | License đã hết hạn |
| 3 | GRACE_PERIOD | License trong thời gian gia hạn |
| 4 | INVALID | License key không hợp lệ |
| 5 | BLOCKED | License bị khóa |
| 6 | QUOTA_EXCEEDED | Đã vượt quota, cần sync |

---

### `WaterMeterSDK.startCameraScan(activity, requestCode)`

Mở màn hình camera quét số đồng hồ với cấu hình mặc định. Kết quả trả về qua `onActivityResult`.

**Tham số:**
- `activity` (Activity) — Activity đang gọi
- `requestCode` (int) — Request code để nhận kết quả trong `onActivityResult`

**Kết quả trong `onActivityResult` (khi `resultCode == RESULT_OK`):**

| Intent extra | Kiểu | Mô tả |
|--------------|------|-------|
| `WaterMeterSDK.EXTRA_RESULT_TEXT` | String | Số đồng hồ đọc được (rỗng nếu thất bại) |
| `WaterMeterSDK.EXTRA_RESULT_CONFIDENCE` | float | Độ tin cậy 0.0–1.0 |
| `WaterMeterSDK.EXTRA_RESULT_IMAGE_PATH` | String | Đường dẫn ảnh đã chụp |

**Lưu ý:** Nếu license chưa kích hoạt, camera vẫn mở nhưng OCR không chạy — chỉ trả về ảnh, không có số.

---

### `WaterMeterSDK.startCameraScan(activity, requestCode, builder)`

Mở màn hình quét với cấu hình tùy chỉnh qua `CameraScanBuilder`.

```java
WaterMeterSDK.startCameraScan(activity, REQUEST_SCAN,
    new WaterMeterSDK.CameraScanBuilder()
        .setTitle("Quét số đồng hồ nước")
        .setShowCloseButton(true)
        .setAutoCloseOnResult(true)
        .setImageMaxWidth(1920)
        .setImageMaxHeight(1080));
```

---

### `WaterMeterSDK.incrementUsage()` / `incrementUsage(amount)`

Tăng quota sử dụng (báo về server chạy nền). **Bình thường không cần gọi** — SDK tự tăng quota sau mỗi lần quét thành công qua màn hình camera. Chỉ dùng khi bạn tự xây luồng xử lý riêng.

## ⚙️ Tuỳ chọn cấu hình (CameraScanBuilder)

| Phương thức | Kiểu | Mặc định | Mô tả |
|-------------|------|----------|-------|
| `setTitle(String)` | String | null | Tiêu đề màn hình quét |
| `setShowCloseButton(boolean)` | boolean | true | Hiện nút đóng (X) |
| `setAutoCloseOnResult(boolean)` | boolean | false | Tự đóng màn hình khi quét thành công |
| `setImageMaxWidth(int)` | number | 0 (ảnh gốc) | Chiều rộng tối đa ảnh lưu (px) |
| `setImageMaxHeight(int)` | number | 0 (ảnh gốc) | Chiều cao tối đa ảnh lưu (px) |

**Lưu ý:** Resize ảnh giữ tỷ lệ. Nếu chỉ định cả width và height, ảnh sẽ fit trong bounds.

## 🔧 Khắc phục sự cố

### Gradle không tải được SDK

```bash
# Kiểm tra đã khai đúng Maven repo:
# maven { url 'https://raw.githubusercontent.com/EOV-Solutions/android-water-meter-sdk/main/repo' }

# Kiểm tra dependency resolve được:
./gradlew :app:dependencies --configuration releaseRuntimeClasspath | grep water-meter
```

### Quét không ra số (chỉ có ảnh, text rỗng)

License chưa được kích hoạt — camera vẫn mở nhưng OCR bị bỏ qua. Kiểm tra:

```java
Log.d("SDK", "Valid: " + WaterMeterSDK.isLicenseValid()
        + ", status: " + WaterMeterSDK.getLicenseStatus()
        + ", message: " + WaterMeterSDK.getStatusMessage());
```

Đảm bảo `initialize()` đã chạy `onSuccess` **trước khi** gọi `startCameraScan()`, và thiết bị có mạng ở lần kích hoạt đầu tiên.

### Lỗi trùng `libc++_shared.so` khi build

```gradle
// app/build.gradle — khi app có thêm native library khác
android {
    packagingOptions {
        pickFirst '**/libc++_shared.so'
    }
}
```

### Crash hoặc không nhận diện sau khi bật minify

Kiểm tra proguard rules đã có:

```proguard
-keep class com.eov.watermeter.** { *; }
-keep class com.baidu.paddle.lite.** { *; }
```

### Lỗi `Manifest merger failed`

Hai nguyên nhân thường gặp:

1. **Xung đột theme** — thông báo lỗi có dạng `Attribute application@theme ... is also present at [com.eov:water-meter-sdk:2.0]`. Thêm `tools:replace="android:theme"` vào tag `<application>` (xem Bước 4 phần Cài đặt).
2. **minSdk thấp hơn 23** — SDK yêu cầu `minSdk >= 23`. Nâng `minSdk` trong `app/build.gradle` lên 23.

### App chạy trên máy ảo x86 bị crash

SDK chỉ hỗ trợ ABI `armeabi-v7a` và `arm64-v8a`. Test trên thiết bị thật hoặc máy ảo ARM64.

## 🔄 Cập nhật version

Khi có bản mới, chỉ cần đổi số version rồi sync lại Gradle:

```gradle
implementation 'com.eov:water-meter-sdk:2.1'
```

Các version cũ vẫn được giữ trong [`repo/`](repo/) để rollback khi cần.

## 📝 Lịch sử thay đổi

### Phiên bản 2.1

- Bỏ `android:theme`, `allowBackup`, `supportsRtl`, `usesCleartextTraffic` khỏi tag `<application>` trong manifest của SDK — không còn xung đột manifest merger với app host, **không cần `tools:replace` nữa**
- `CameraSettingsActivity` khai theme riêng (giữ nguyên giao diện)
- Không còn âm thầm bật cleartext traffic cho app host (license server dùng HTTPS)

### Phiên bản 2.0
cccc
- Model AI đóng gói sẵn trong AAR
- Điều khiển flash và zoom
- Tự động chụp khi độ tin cậy cao
- Quản lý license tích hợp (kích hoạt, trạng thái, quota)
- Tùy chỉnh màn hình quét qua `CameraScanBuilder`
- Lưu ảnh và trả về đường dẫn, hỗ trợ resize
- Tương thích 16KB page size (Google Play requirement)
- Hỗ trợ Android 6.0+ (API 23), ABI `armeabi-v7a` / `arm64-v8a`

## 📄 Giấy phép

SDK thương mại của EOV Solutions — sử dụng theo hợp đồng license. Liên hệ EOV Solutions để được cấp license key.

## 👥 Tác giả

EOV Solutions

---

*Làm bởi EOV Solutions*
