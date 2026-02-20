## API.md — Image Processor gRPC API (Synchronous)

### Overview

This service provides synchronous image processing via a single gRPC call. The client sends an image file (bytes) and an ordered list of operations to apply. The server applies operations sequentially (cumulative transforms) and returns the resulting image bytes and an optional thumbnail.

### Supported image formats

Minimum supported input formats:

* PNG
* JPG/JPEG
* GIF

Output format behavior:

* Default: same format as input (unless the client requests a different output format)

### RPC Service

**Service:** `ImageProcessor`  
**Method:** `ProcessImage(ProcessImageRequest) returns (ProcessImageResponse)`  
**Call style:** Unary RPC (single request → single response), synchronous.

---

## Request / Response Schemas

### ProcessImageRequest

| Field              | Type                 | Required | Description                                                                                    |
| ------------------ | -------------------- | -------- | ---------------------------------------------------------------------------------------------- |
| `image_bytes`      | `bytes`              | Yes      | Raw bytes of the input image file.                                                             |
| `input_format`     | `string`             | Yes      | Format hint: `"png"`, `"jpeg"`, `"gif"`.                                                       |
| `operations`       | `repeated Operation` | Yes      | Ordered list of operations to apply. Applied sequentially.                                     |
| `output_format`    | `string`             | No       | Optional output override: `"png"` or `"jpeg"`. If empty, use same-as-input.                    |
| `return_thumbnail` | `bool`               | No       | If true, return a single thumbnail if any `Thumbnail` op is present (last thumbnail wins).    |
| `rotate_fill_rgb`  | `Rgb`                | No       | Fill color for empty canvas space created by arbitrary rotation. Default: white (255,255,255). |

> **Limits (chosen; document your values):**  
> Max file size: `10 MB`  
> Max dimensions: `8000 x 8000`  
> Max operations per request: `32`

### ProcessImageResponse

| Field                   | Type        | Present  | Description                                                                                  |
| ----------------------- | ----------- | -------- | -------------------------------------------------------------------------------------------- |
| `result_image_bytes`    | `bytes`     | Always   | Output image bytes after all operations.                                                     |
| `result_format`         | `string`    | Always   | `"png"` / `"jpeg"` / `"gif"` (depending on output rules).                                    |
| `thumbnail_image_bytes` | `bytes`     | Optional | Only present if `return_thumbnail=true` and at least one `Thumbnail` operation was included. |
| `thumbnail_format`      | `string`    | Optional | Thumbnail output format (recommend fixed `"png"`).                                           |
| `input_info`            | `ImageInfo` | Always   | Input metadata (width/height/byte_size).                                                     |
| `result_info`           | `ImageInfo` | Always   | Output metadata (width/height/byte_size).                                                    |
| `thumbnail_info`        | `ImageInfo` | Optional | Thumbnail metadata (always 300x300).                                                         |

---

## Operation “Little Language”

The `operations[]` field is an ordered list of `Operation` messages. Each `Operation` uses a `oneof` so **exactly one operation type** is set per list element.

Operations run in order:

* Output of operation *i* is the input to operation *i+1*
* Rotations are cumulative (e.g., rotate 10° then rotate right = 100°)

---

## Operation Parameters (what your professor asked for)

### FlipHorizontal

* **Params:** none  
* **Effect:** mirror left ↔ right

### FlipVertical

* **Params:** none  
* **Effect:** mirror top ↔ bottom

### RotateLeft

* **Params:** none  
* **Effect:** rotate −90° (cumulative)

### RotateRight

* **Params:** none  
* **Effect:** rotate +90° (cumulative)

### RotateDegrees

* **Params:**
  * `degrees` (int): whole degrees, may be negative
* **Rules:**
  * canvas expands so no image content is lost
  * empty canvas area filled with `rotate_fill_rgb` (default white)
* **Errors (fail request):**
  * if result exceeds max pixel constraints

### Grayscale

* **Params:** none  
* **Effect:** convert to grayscale (preserve alpha when possible)

### ResizePercent

* **Params:**
  * `percent` (int): absolute scale factor where `100` = original size
* **Rules:**
  * uniform scaling (aspect ratio preserved)
  * whole percents only
* **Errors (fail request):**
  * `percent < 1`
  * result exceeds max pixel constraints
  * result rounds to 0 width or 0 height

### Thumbnail

* **Params:** none (fixed requirements)
* **Rules:**
  * output is always **300×300**
  * aspect ratio preserved, padded (no crop, no stretch)
  * **last thumbnail op wins** (only one thumbnail returned)

---

## Error Handling

On error, the RPC fails with a gRPC status code and a message that includes which operation failed.

Recommended codes:

* `INVALID_ARGUMENT`: bad input format, invalid operation parameters, violates limits
* `RESOURCE_EXHAUSTED`: file too large / pixel limit exceeded
* `INTERNAL`: unexpected library error

Recommended error message format:  
`op_index=<n> op=<OP_NAME> reason=<description>`

Example:  
`INVALID_ARGUMENT: op_index=2 op=RESIZE_PERCENT reason=result exceeds max 8000x8000`

---

## Example API Calls and Responses

### Example 1 — Basic transformation pipeline

**Goal:** Rotate 45°, convert to grayscale, resize to 50%, and generate a thumbnail.

#### Example Request (conceptual)

```
ProcessImageRequest {
  image_bytes: <binary image data>
  input_format: "jpeg"
  output_format: ""            // empty = same as input
  return_thumbnail: true
  rotate_fill_rgb: { r:255 g:255 b:255 }
  operations: [
    { rotate_degrees: { degrees: 45 } },
    { grayscale: {} },
    { resize_percent: { percent: 50 } },
    { thumbnail: {} }
  ]
}
```

#### Example Response (conceptual)

```
ProcessImageResponse {
  result_image_bytes: <binary image data>
  result_format: "jpeg"
  thumbnail_image_bytes: <binary image data>
  thumbnail_format: "png"
  input_info:  { width: 1280 height: 960  byte_size: 345678 }
  result_info: { width:  905 height: 905  byte_size: 210123 }
  thumbnail_info: { width: 300 height: 300 byte_size: 45678 }
}
```

### Example 2 — Minimal pipeline (flip only)

**Goal:** Flip image horizontally.

#### Example Request (conceptual)

```
ProcessImageRequest {
  image_bytes: <binary image data>
  input_format: "png"
  return_thumbnail: false
  operations: [
    { flip_horizontal: {} }
  ]
}
```

#### Example Response (conceptual)

```
ProcessImageResponse {
  result_image_bytes: <binary image data>
  result_format: "png"
  input_info:  { width: 800 height: 600 byte_size: 120345 }
  result_info: { width: 800 height: 600 byte_size: 120220 }
}
```

### Example 3 — Error case (invalid resize)

**Scenario:** Resize percent is invalid.

#### Example Request (conceptual)

```
ProcessImageRequest {
  image_bytes: <binary image data>
  input_format: "jpeg"
  operations: [
    { resize_percent: { percent: 0 } }
  ]
}
```

#### Example Error (gRPC status)

```
INVALID_ARGUMENT:
op_index=0 op=RESIZE_PERCENT reason=percent must be >= 1
```
