# OpenCamera - Extended Formats Edition

This repository is a friendly fork of the main [OpenCamera](https://sourceforge.net/projects/opencamera/) project, introducing support for modern, high-efficiency image formats. The goal is to merge into OpenCamera when it is good enough.

## Divergence from the Main Repository

This version diverges from the upstream OpenCamera repository by adding capabilities to capture and save photos in the following formats:
*   **JPEG XL (JXL) - Fast**
*   **JPEG XL (JXL) - High Compression**
*   **AVIF**
*   **HEIC**

### Key Modifications:
1.  **Dependencies:** Added external libraries to `app/build.gradle` to handle the new encodings:
    *   `io.github.awxkee:jxl-coder` for JPEG XL encoding.
    *   `com.github.awxkee:avif-coder` as a software fallback for AVIF encoding.
    *   `androidx.heifwriter:heifwriter` for native HEIC (and hardware-accelerated AVIF) encoding.
2.  **Manifest:** Removed `android:extractNativeLibs="true"` from `AndroidManifest.xml`. By default in AGP 8.3+, native libraries are packaged uncompressed and natively 16KB aligned, which fixes 16KB alignment issues on newer Android 15 devices while still allowing the C++ encoders to function correctly.
3.  **Image Saver (`ImageSaver.java`):** Completely overhauled the image saving pipeline to intercept the camera's raw byte output, decode it into a `Bitmap`, and re-encode it using the selected modern format (JxlCoder, HeifCoder, or HeifWriter) before writing it to storage.
4.  **Storage Utilities (`StorageUtils.java`):** Registered the new MIME types (`image/jxl`, `image/avif`, `image/heic`) to ensure proper integration with the Android MediaStore and gallery applications.

---

## Technical Limitations Regarding EXIF Metadata

While integrating these new formats, several technical limitations were encountered specifically regarding the preservation of EXIF metadata (such as GPS location, timestamps, and camera settings). 

In standard OpenCamera operation (JPEG), EXIF data is managed via Android's `androidx.exifinterface.media.ExifInterface`. However, this interface and the underlying encoding libraries present challenges for newer formats.

### 1. HEIC - Fully Supported ✅
HEIC successfully preserves all EXIF data, including GPS location. 
**Implementation:** OpenCamera intercepts the original JPEG EXIF bytes from the camera hardware. It writes these bytes to a temporary `.jpg` file, modifies them using the standard `ExifInterface` (to add GPS, etc.), reads the modified bytes back, and explicitly injects them into the HEIC container using `androidx.heifwriter.HeifWriter.addExifData()`.

### 2. JPEG XL (JXL) - Fully Supported ✅
Images saved in JXL (both Fast and High Compression variants) successfully preserve all EXIF metadata, including GPS location.
**Implementation:** The `io.github.awxkee:jxl-coder` library encodes a raw JXL codestream directly from an Android `Bitmap`. To preserve metadata, OpenCamera creates a temporary `.jpg` file, modifies it using `ExifInterface` (ensuring GPS location is added), and extracts the raw TIFF EXIF payload. It then manually constructs an ISO Base Media File Format (BMFF) container, wrapping both the extracted EXIF payload and the JXL codestream into a fully compliant `.jxl` file.

### 3. AVIF - EXIF Lost ❌
Images saved in AVIF format currently lose all EXIF metadata on most devices.
**Technical Details:**
*   **Hardware Path (Failed):** The ideal implementation would use Android's `androidx.heifwriter.AvifWriter`, which supports `addExifData()` exactly like the HEIC implementation. However, `AvifWriter` requires a compatible hardware AV1 encoder. On devices without one (e.g., Pixel 5), `AvifWriter.Builder.build()` crashes with an `IllegalArgumentException` (*"Cannot find capable encoder with the current encoder preference"*).
*   **Software Fallback (Current):** To prevent crashes and ensure the photo is saved, the app falls back to using `com.radzivon.bartoshyk.avif.coder.HeifCoder`. Like the JXL encoder, this software encoder takes a `Bitmap` but lacks a mechanism to inject raw EXIF bytes.
*   **Post-Processing (Failed):** Similar to JXL, Android's `ExifInterface` refuses to write to `.avif` files, throwing the same format-restriction `IOException` if attempted post-save.
