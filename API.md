# API.md — Image Processor gRPC API (Synchronous)

## Overview

This service provides synchronous image processing via a single gRPC call. The client sends an image file (bytes) and an ordered list of operations to apply. The server applies operations sequentially (cumulative transforms) and returns the resulting image bytes and an optional thumbnail.

---

## Supported Image Formats

Minimum supported input formats:

* PNG
* JPG/JPEG
* GIF

### Output behavior

* The result image format is always the **same as the input format**
* Thumbnail images are always returned as **PNG**
* Arbitrary rotation expands the canvas and fills empty space with **white (RGB 255,255,255)**

---

## RPC Service

**Service:** `ImageProcessor`
**Method:** `ProcessImage(ProcessImageRequest) returns (ProcessImageResponse)`
**Call style:** Unary gRPC call (single request → single response), synchronous.

---

# Request / Response Schemas

## ProcessImageRequest

| Field              | Type                 | Required | Description                                                                                       |
| ------------------ | -------------------- | -------- | ------------------------------------------------------------------------------------------------- |
| `image_bytes`      | `bytes`              | Yes      | Raw bytes of the input image file.                                                                |
| `input_format`     | `string`             | Yes      | Format hint: `"png"`, `"jpeg"`, `"gif"`.                                                          |
| `operations`       | `repeated Operation` | Yes      | Ordered list of operations to apply. Applied sequentially.                                        |
| `return_thumbnail` | `bool`               | No       | If true, return a single thumbnail if any `Thumbnail` operation is present (last thumbnail wins). |

### Limits (chosen)

* **Max file size:** 10 MB
* **Max dimensions:** 8000 × 8000
* **Max operations per request:** 32

---

## ProcessImageResponse

| Field                   | Type     | Present  | Description                                                                                  |
| ----------------------- | -------- | -------- | -------------------------------------------------------------------------------------------- |
| `result_image_bytes`    | `bytes`  | Always   | Output image bytes after all operations.                                                     |
| `result_format`         | `string` | Always   | `"png"` or `"jpeg"` or `"gif"` (matches input format).                                       |
| `thumbnail_image_bytes` | `bytes`  | Optional | Present only if `return_thumbnail=true` and at least one `Thumbnail` operation was included. |

---

# Operations

The `operations[]` field is an ordered list of `Operation` messages. Each `Operation` uses a `oneof`, so **exactly one operation type** is set per list element.

Operations execute sequentially:

* Output of operation *i* becomes input to operation *i+1*
* Rotations are cumulative (e.g., rotate 10° then rotate right = 100°)

---

# Operation Parameters

## FlipHorizontal

* **Parameters:** none
* **Effect:** Mirrors image left ↔ right

---

## FlipVertical

* **Parameters:** none
* **Effect:** Mirrors image top ↔ bottom

---

## RotateLeft

* **Parameters:** none
* **Effect:** Rotates image −90° (cumulative)

---

## RotateRight

* **Parameters:** none
* **Effect:** Rotates image +90° (cumulative)

---

## RotateDegrees

**Parameters**

* `degrees` (int): whole degrees, may be negative

**Behavior**

* Canvas expands so no image content is lost
* Empty canvas areas are filled with **white (RGB 255,255,255)**

**Error conditions (fail request)**

* Result exceeds maximum pixel constraints

---

## Grayscale

* **Parameters:** none
* **Effect:** Converts image to grayscale (preserving alpha when supported)

---

## ResizePercent

**Parameters**

* `percent` (int): absolute scale factor where `100` = original size

**Behavior**

* Uniform scaling (aspect ratio preserved)
* Whole percentages only

**Error conditions (fail request)**

* `percent < 1`
* Result exceeds max pixel constraints
* Result rounds to zero width or height

---

## Thumbnail

**Parameters:** none (fixed behavior)

**Behavior**

* Output is always **300px × 300px**
* Aspect ratio preserved with padding (no crop, no stretch)
* **Last thumbnail operation wins** (only one thumbnail returned)

---

# Error Handling

On error, the RPC fails with a gRPC status code and a descriptive message that identifies the failing operation.

## Recommended Status Codes

* `INVALID_ARGUMENT` — invalid input, invalid parameters, constraint violation
* `RESOURCE_EXHAUSTED` — file too large or exceeds pixel limits
* `INTERNAL` — unexpected image processing failure

## Error Message Format

```
op_index=<n> op=<OP_NAME> reason=<description>
```

### Example

```
INVALID_ARGUMENT:
op_index=2 op=RESIZE_PERCENT reason=result exceeds max 8000x8000
```

---

# Example API Calls and Responses

## Example 1 — Full Transformation Pipeline

**Goal:** Rotate 45°, convert to grayscale, resize to 50%, and generate a thumbnail.

### Example Request (conceptual)

```
ProcessImageRequest {
  image_bytes: <binary image data>
  input_format: "jpeg"
  return_thumbnail: true
  operations: [
    { rotate_degrees: { degrees: 45 } },
    { grayscale: {} },
    { resize_percent: { percent: 50 } },
    { thumbnail: {} }
  ]
}
```

### Example Response (conceptual)

```
ProcessImageResponse {
  result_image_bytes: <binary image data>
  result_format: "jpeg"
  thumbnail_image_bytes: <binary image data>
}
```

---

## Example 2 — Minimal Pipeline

**Goal:** Flip image horizontally.

### Example Request

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

### Example Response

```
ProcessImageResponse {
  result_image_bytes: <binary image data>
  result_format: "png"
}
```

---

## Example 3 — Error Case

**Scenario:** Invalid resize percentage.

### Example Request

```
ProcessImageRequest {
  image_bytes: <binary image data>
  input_format: "jpeg"
  operations: [
    { resize_percent: { percent: 0 } }
  ]
}
```

### Example Error (gRPC status)

```
INVALID_ARGUMENT:
op_index=0 op=RESIZE_PERCENT reason=percent must be >= 1
```
