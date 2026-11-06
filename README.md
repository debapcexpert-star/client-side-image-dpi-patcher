# Client-Side Image DPI Patcher (Vanilla JS)

This is a lightweight, zero-dependency JavaScript function that modifies the JFIF APP0 header of an image in-memory to set exact DPI metadata (e.g., 200 DPI, 300 DPI, 600 DPI) entirely in the browser.

## Why this exists
Standard web-based image compressors and HTML5 `<canvas>` exports strip DPI metadata. This causes strict government portals (like US Visas, Canada PR, or Indian banking exams) to instantly reject the photos. This script patches the binary data directly to satisfy strict validators without requiring backend processing.

## Live Production Demo
I use this exact script in production at **[DocPixelKB](https://docpixelkb.com)**[cite: 1] to process thousands of strict government exam and passport photos locally, without uploading user biometrics to a server[cite: 1]. 

## The Code

```javascript
/**
 * Injects or updates the JFIF APP0 marker to set the exact DPI of a JPEG image.
 * 
 * @param {Uint8Array} binary - Raw JPEG binary data.
 * @param {number} dpi - Target DPI (e.g., 300 or 600).
 * @returns {Uint8Array} - JPEG binary with patched JFIF APP0 DPI metadata.
 */
function patchJfifDpi(binary, dpi) {
  // Clamp target DPI to safe positive integer
  const safeDpi = Math.max(72, Math.min(1200, Math.round(dpi)));
  const dpiHigh = (safeDpi >> 8) & 0xFF;
  const dpiLow = safeDpi & 0xFF;

  // Verify JPEG SOI (Start of Image) marker: 0xFF, 0xD8
  if (binary.length < 4 || binary[0] !== 0xFF || binary[1] !== 0xD8) {
    return binary;
  }

  // Check if JFIF APP0 marker exists at index 2
  const isApp0 = binary[2] === 0xFF && binary[3] === 0xE0;
  const isJfif =
    binary.length >= 18 &&
    binary[6] === 0x4A && // 'J'
    binary[7] === 0x46 && // 'F'
    binary[8] === 0x49 && // 'I'
    binary[9] === 0x46 && // 'F'
    binary[10] === 0x00; // '\0'

  if (isApp0 && isJfif) {
    // Clone and patch in-place
    const patched = new Uint8Array(binary);
    patched[13] = 0x01;    // Density unit: 1 = Dots Per Inch (DPI)
    patched[14] = dpiHigh; // X density (high byte)
    patched[15] = dpiLow;  // X density (low byte)
    patched[16] = dpiHigh; // Y density (high byte)
    patched[17] = dpiLow;  // Y density (low byte)
    return patched;
  }

  // If JFIF APP0 is missing (standard HTML5 Canvas output), construct 18-byte APP0 segment
  const jfifSegment = new Uint8Array([
    0xFF, 0xE0,                   // APP0 marker
    0x00, 0x10,                   // Segment length: 16 bytes (excluding marker)
    0x4A, 0x46, 0x49, 0x46, 0x00, // Identifier: "JFIF\0"
    0x01, 0x01,                   // Version: 1.1
    0x01,                         // Units: 1 = dots per inch (DPI)
    dpiHigh, dpiLow,              // X density (e.g. 300 = 0x01, 0x2C)
    dpiHigh, dpiLow,              // Y density
    0x00, 0x00                    // Thumbnail dimensions: 0 x 0
  ]);

  // Insert JFIF APP0 segment immediately after SOI (0xFF, 0xD8)
  const output = new Uint8Array(binary.length + jfifSegment.length);
  output[0] = 0xFF;
  output[1] = 0xD8;
  output.set(jfifSegment, 2);
  output.set(binary.subarray(2), 2 + jfifSegment.length);

  return output;
}

// Usage with HTML5 Canvas / Blob:
// Convert Canvas blob to Uint8Array, patch DPI, and create download Blob
/*
canvas.toBlob(async (blob) => {
  const buffer = await blob.arrayBuffer();
  const inputBinary = new Uint8Array(buffer);
  
  // Set to 300 or 600 DPI
  const patchedBinary = patchJfifDpi(inputBinary, 300);

  const outputBlob = new Blob([patchedBinary], { type: 'image/jpeg' });
  const downloadUrl = URL.createObjectURL(outputBlob);
}, 'image/jpeg', 0.92);
*/
