# MinilibX (MLX) — Complete Python Documentation
### Version 2.2 — mlx_CLXV — Linux/XCB/Vulkan

---

## Table of Contents

1. [What is MLX?](#1-what-is-mlx)
2. [How it Works (Architecture)](#2-how-it-works-architecture)
3. [Getting Started](#3-getting-started)
4. [Initialization & Teardown](#4-initialization--teardown)
5. [Windows](#5-windows)
6. [Drawing Directly](#6-drawing-directly)
7. [Images](#7-images)
8. [Loading Image Files](#8-loading-image-files)
9. [The Event Loop](#9-the-event-loop)
10. [Keyboard Hooks](#10-keyboard-hooks)
11. [Mouse Hooks](#11-mouse-hooks)
12. [Expose Hook](#12-expose-hook)
13. [Loop Hook](#13-loop-hook)
14. [Generic Hook — mlx_hook](#14-generic-hook--mlx_hook)
15. [Mouse Utilities](#15-mouse-utilities)
16. [Keyboard Utilities](#16-keyboard-utilities)
17. [Screen Info](#17-screen-info)
18. [Sync & Flush](#18-sync--flush)
19. [Color System](#19-color-system)
20. [Practical Recipes](#20-practical-recipes)
21. [Limitations & Known Issues](#21-limitations--known-issues)
22. [Quick Reference Card](#22-quick-reference-card)

---

## 1. What is MLX?

**MinilibX** (MLX) is a minimal graphical library created by Olivier Crouzet for 42 school students. Its entire purpose is to let you open a window, draw pixels in it, load images, and react to keyboard and mouse events — without needing to know Vulkan, X11, or any other low-level graphics system.

It is intentionally small. It does **not** provide:
- Text rendering beyond a basic fixed-size bitmap font
- Audio
- 3D or GPU shaders (the GPU is used internally, but not exposed)
- Menus, widgets, or UI components
- Networking or remote display

Everything beyond these basics is for you to build.

The Python version (`mlx_CLXV 2.2`) is a `ctypes` wrapper around the C library (`libmlx.so`). Every Python method maps directly to a C function with the same name.

---

## 2. How it Works (Architecture)

```
Your Python Code
      │
      ▼
  Mlx class (mlx.py)        ← Python ctypes wrapper
      │
      ▼
  libmlx.so                 ← compiled C library
      │
      ├── XCB backend       ← talks to the X server (window, events)
      └── Vulkan backend    ← GPU drawing (pixels, images, text)
```

**The main loop** (`mlx_loop`) is an infinite event loop inside the C library. It:
- Waits for events (key presses, mouse clicks, window events)
- Calls your registered Python callback functions when those events occur
- Calls your loop hook function on every iteration when no other event is pending

Your program must register hooks **before** calling `mlx_loop`. Once inside the loop, your code only runs when a hook fires.

---

## 3. Getting Started

Minimum working program:

```python
from mlx import Mlx
import os

def on_key(key, param):
    if key == 65307:  # ESC
        os._exit(0)

mlx = Mlx()
mlx_ptr = mlx.mlx_init()
win_ptr = mlx.mlx_new_window(mlx_ptr, 800, 600, "My Window")

mlx.mlx_key_hook(win_ptr, on_key, None)
mlx.mlx_hook(win_ptr, 33, 0, lambda p: os._exit(0), None)  # close button

mlx.mlx_loop(mlx_ptr)
```

---

## 4. Initialization & Teardown

### `mlx_init()`

```python
mlx_ptr = mlx.mlx_init()
```

**Must be called first**, before anything else. Creates the connection to the display system (XCB + Vulkan).

- **Returns:** connection identifier (`void *`) — pass this to most other functions
- **Returns `None`** on failure

---

### `mlx_release(mlx_ptr)`

```python
mlx.mlx_release(mlx_ptr)
```

Disconnects from the display and frees all internal resources. Call this at the end of your program after the loop exits.

- **Returns:** `0` on success

---

## 5. Windows

### `mlx_new_window(mlx_ptr, width, height, title)`

```python
win_ptr = mlx.mlx_new_window(mlx_ptr, 1600, 900, "A-MAZE-ING")
```

Creates a new window on screen. MLX can manage **multiple windows simultaneously** — each call creates a separate window.

- **`width`**: window width in pixels
- **`height`**: window height in pixels
- **`title`**: text shown in the window's title bar (string, auto-encoded to UTF-8)
- **Returns:** window identifier — pass this to drawing and hook functions
- **Returns `None`** on failure

> Origin `(0, 0)` is the **top-left corner**. X increases right, Y increases downward.

---

### `mlx_clear_window(mlx_ptr, win_ptr)`

```python
mlx.mlx_clear_window(mlx_ptr, win_ptr)
```

Fills the entire window with **black** (`0x000000`). Use this to erase everything before redrawing each frame.

- **Returns:** `0` on success

---

### `mlx_destroy_window(mlx_ptr, win_ptr)`

```python
mlx.mlx_destroy_window(mlx_ptr, win_ptr)
```

Closes and destroys the window. The window identifier becomes invalid after this call. Useful when managing multiple windows and you want to close just one.

- **Returns:** `0` on success

---

## 6. Drawing Directly

### `mlx_pixel_put(mlx_ptr, win_ptr, x, y, color)`

```python
mlx.mlx_pixel_put(mlx_ptr, win_ptr, 100, 200, 0xFFFFFFFF)
```

Draws a **single pixel** at position `(x, y)` with the given color. This is the most basic drawing primitive.

- **`x`, `y`**: pixel coordinates (top-left origin)
- **`color`**: unsigned int, format `0xAARRGGBB` (see Color section)
- **Returns:** `0` on success

> **Warning:** `mlx_pixel_put` is **slow** because it sends each pixel individually to the GPU. For anything beyond a few pixels, use images instead.

---

### `mlx_string_put(mlx_ptr, win_ptr, x, y, color, string)`

```python
mlx.mlx_string_put(mlx_ptr, win_ptr, 100, 50, 0xFFFFFFFF, "Hello World")
```

Renders a text string at position `(x, y)` using MLX's **built-in bitmap font**.

- **`x`, `y`**: position of the top-left of the first character
- **`color`**: text color as unsigned int (only RGB used, alpha is forced to 255)
- **`string`**: the text to display (auto-encoded to UTF-8)
- **Returns:** `0` on success

**Font details:**
- Fixed-size bitmap font (same font on all platforms using this MLX version)
- Supports ASCII characters 32–127 (space through `~`)
- Characters outside that range display as a placeholder
- You **cannot** change the font size or font face
- Approximately 7–8 pixels wide per character, ~13 pixels tall

**What you can do with text:**
- Change color to anything (white, yellow, red, etc.)
- Position it anywhere in the window
- Draw it on top of images (images drawn after will cover it)
- Create simple labels, counters, status text

**What you cannot do:**
- Change font size
- Use bold/italic
- Get the pixel width of a string (estimate ~7–8 px per char)

---

## 7. Images

Images are the **recommended way to draw** in MLX. Instead of sending pixel-by-pixel to the GPU, you build an image in memory then send the whole thing at once.

### `mlx_new_image(mlx_ptr, width, height)`

```python
img_ptr = mlx.mlx_new_image(mlx_ptr, 200, 200)
```

Creates a blank image in memory of the given size.

- **Returns:** image identifier, or `None` on failure

---

### `mlx_get_data_addr(img_ptr)`

```python
data, bits_per_pixel, size_line, img_format = mlx.mlx_get_data_addr(img_ptr)
```

Returns a **writable memoryview** of the image's raw pixel data, plus metadata. This is the key function for drawing into images.

- **`data`**: `memoryview` (cast to `'B'`) — writable byte buffer of the image
- **`bits_per_pixel`**: how many bits represent one pixel (typically `32`)
- **`size_line`**: number of **bytes** per row of the image
- **`img_format`**:
  - `0` = `B8G8R8A8` byte order
  - `1` = `A8R8G8B8` byte order

**Writing a single pixel into an image:**

```python
def put_pixel(data, size_line, x, y, color_bytes):
    offset = y * size_line + x * 4
    data[offset:offset + 4] = color_bytes

# Example: white pixel at (10, 20)
put_pixel(data, size_line, 10, 20, (0xFF, 0xFF, 0xFF, 0xFF))
```

**Filling the whole image with one color:**

```python
color = (0xFF, 0x00, 0x00, 0xFF)  # red, fully opaque
for offset in range(0, size_line * height, 4):
    data[offset:offset + 4] = color
```

---

### `mlx_put_image_to_window(mlx_ptr, win_ptr, img_ptr, x, y)`

```python
mlx.mlx_put_image_to_window(mlx_ptr, win_ptr, img_ptr, 0, 0)
```

Renders an image onto the window at position `(x, y)`. The image's top-left corner lands at `(x, y)`. Images drawn later will appear on top of earlier ones.

- **`x`, `y`**: can be negative (image partially off-screen is fine)
- **Returns:** `0` on success

> This is fast — it sends the whole image buffer to the GPU in one call.

---

### `mlx_destroy_image(mlx_ptr, img_ptr)`

```python
mlx.mlx_destroy_image(mlx_ptr, img_ptr)
```

Frees the memory for an image. Always destroy images you no longer need to avoid memory leaks.

- **Returns:** `0` on success

---

## 8. Loading Image Files

### `mlx_xpm_file_to_image(mlx_ptr, filename)`

```python
img_ptr, width, height = mlx.mlx_xpm_file_to_image(mlx_ptr, "assets/wall.xpm")
```

Loads an **XPM file** and creates an image from it. **XPM is the recommended format** for this MLX version — it is the most reliable loader.

- **`filename`**: path to the `.xpm` file (string, auto-encoded)
- **Returns:** tuple `(img_ptr, width, height)`
  - `img_ptr` is `None` on failure
- **Supports transparency** in XPM

> XPM files can be generated from PNG using ImageMagick:
> ```bash
> magick input.png output.xpm
> ```

---

### `mlx_png_file_to_image(mlx_ptr, filename)`

```python
img_ptr, width, height = mlx.mlx_png_file_to_image(mlx_ptr, "assets/wall.png")
```

Loads a **PNG file** and creates an image from it.

- **Returns:** tuple `(img_ptr, width, height)`
  - `img_ptr` is `None` on failure

> **Warning:** The PNG loader in this MLX version is incomplete and unreliable on some systems/PNG types. Prefer XPM for production use — convert PNGs to XPM first with ImageMagick.

---

## 9. The Event Loop

### `mlx_loop(mlx_ptr)`

```python
mlx.mlx_loop(mlx_ptr)  # ← program runs here until exit
```

Starts the **infinite event loop**. This call **never returns** under normal operation. The program now runs entirely through callbacks registered with the various hook functions.

- Must be called **after** all hooks are registered
- All windows, images, and hooks must be set up **before** this call
- Returns only if `mlx_loop_exit` is called from within a callback

> **Note for Python:** You cannot catch `Ctrl+C` (SIGINT) while inside `mlx_loop`. Use `Ctrl+\` (SIGQUIT) to force-kill the process, or call `os._exit(0)` from within a key hook.

---

### `mlx_loop_exit(mlx_ptr)`

```python
mlx.mlx_loop_exit(mlx_ptr)
```

Signals the event loop to stop. `mlx_loop` will return after this is called. Use this for clean shutdown instead of `os._exit(0)` when you want code to run after the loop (resource cleanup, etc.).

- **Returns:** `0` on success

---

## 10. Keyboard Hooks

### `mlx_key_hook(win_ptr, callback, param)`

```python
def on_key(keycode, param):
    print(f"Key released: {keycode}")

mlx.mlx_key_hook(win_ptr, on_key, None)
```

Registers a function to call when a **key is released** (KeyRelease event, not KeyPress).

- **`callback`**: function with signature `(keycode: int, param: any) -> None`
- **`param`**: any Python object, passed back to the callback unchanged
- **Returns:** `0` on success

**To disable the hook:**
```python
mlx.mlx_key_hook(win_ptr, None, None)
```

**Common key codes (Linux/XCB):**

| Key       | Code  | Key      | Code  |
|-----------|-------|----------|-------|
| ESC       | 65307 | Space    | 32    |
| Enter     | 65293 | Tab      | 65289 |
| Up arrow  | 65362 | Down     | 65364 |
| Left      | 65361 | Right    | 65363 |
| `a`       | 97    | `z`      | 122   |
| `0`       | 48    | `9`      | 57    |
| `1`       | 49    | `2`      | 50    |
| F1        | 65470 | F12      | 65481 |
| Backspace | 65288 | Delete   | 65535 |
| Shift (L) | 65505 | Ctrl (L) | 65507 |

> Find any key code by printing it: `print(f"Key: {keycode}")`

---

## 11. Mouse Hooks

### `mlx_mouse_hook(win_ptr, callback, param)`

```python
def on_mouse(button, x, y, param):
    print(f"Button {button} clicked at ({x}, {y})")

mlx.mlx_mouse_hook(win_ptr, on_mouse, None)
```

Registers a function to call when a **mouse button is pressed**.

- **`callback`**: function with signature `(button: int, x: int, y: int, param: any) -> None`
  - `button`: which button was clicked (see below)
  - `x`, `y`: coordinates of the click **inside the window** (top-left origin)
- **`param`**: any Python object, passed back unchanged
- **Returns:** `0` on success

**Mouse button codes:**

| Button        | Code |
|---------------|------|
| Left click    | 1    |
| Middle click  | 2    |
| Right click   | 3    |
| Scroll up     | 4    |
| Scroll down   | 5    |

**To disable the hook:**
```python
mlx.mlx_mouse_hook(win_ptr, None, None)
```

**Hit-test example (clickable button area):**

```python
BTN_X, BTN_Y, BTN_W, BTN_H = 100, 200, 240, 55

def on_mouse(button, x, y, param):
    if button == 1:
        if BTN_X <= x <= BTN_X + BTN_W and BTN_Y <= y <= BTN_Y + BTN_H:
            print("Button clicked!")

mlx.mlx_mouse_hook(win_ptr, on_mouse, None)
```

---

## 12. Expose Hook

### `mlx_expose_hook(win_ptr, callback, param)`

```python
def on_expose(param):
    state.needs_redraw = True

mlx.mlx_expose_hook(win_ptr, on_expose, None)
```

Registers a function to call when a window **needs to be redrawn** (e.g., after being uncovered by another window or after alt-tab).

- **`callback`**: function with signature `(param: any) -> None`
- **Returns:** `0` on success

> **Important:** On modern Linux with Wayland or X11 with compositing, this event may only fire **once** at program start (or never). The X server often saves the window content itself. Always re-draw from your `loop_hook` using a dirty flag pattern instead of relying solely on expose.

---

## 13. Loop Hook

### `mlx_loop_hook(mlx_ptr, callback, param)`

```python
def on_loop(param):
    if state.needs_redraw:
        mlx.mlx_clear_window(mlx_ptr, win_ptr)
        draw_everything()
        state.needs_redraw = False

mlx.mlx_loop_hook(mlx_ptr, on_loop, None)
```

Registers a function that is called **on every iteration of the event loop** when no other event is pending. This is the primary place for your rendering logic.

- **`callback`**: function with signature `(param: any) -> None`
- **Not bound to a window** — bound to the `mlx_ptr` connection
- **Returns:** `0` on success

**Recommended pattern — dirty flag:**

```python
state.needs_redraw = True  # set to True whenever something changes

def on_loop(param):
    if not state.needs_redraw:
        return  # nothing to do, skip this frame
    mlx.mlx_clear_window(mlx_ptr, win_ptr)
    # ... draw ...
    state.needs_redraw = False
```

This avoids redrawing 60+ times per second when nothing has changed.

---

## 14. Generic Hook — `mlx_hook`

### `mlx_hook(win_ptr, x_event, x_mask, callback, param)`

```python
mlx.mlx_hook(win_ptr, 33, 0, on_close, None)
```

The **most powerful hook** — lets you listen to any X11 event type, not just the three covered by the convenience hooks.

- **`x_event`**: X11 event number (see table below)
- **`x_mask`**: X11 event mask (usually `0` for modern MLX; masks are often ignored)
- **`callback`**: function — signature depends on the event type (see below)
- **Returns:** `0` on success

**X11 Event Numbers (most useful):**

| x_event | Name            | When it fires                          | Callback signature              |
|---------|-----------------|----------------------------------------|---------------------------------|
| `2`     | KeyPress        | Key held down (before release)         | `(keycode, param)`              |
| `3`     | KeyRelease      | Key released (same as key_hook)        | `(keycode, param)`              |
| `4`     | ButtonPress     | Mouse button pressed                   | `(button, x, y, param)`         |
| `5`     | ButtonRelease   | Mouse button released                  | `(button, x, y, param)`         |
| `6`     | MotionNotify    | Mouse moved while button held          | `(x, y, param)`                 |
| `7`     | EnterNotify     | Mouse cursor enters the window         | `(param)`                       |
| `8`     | LeaveNotify     | Mouse cursor leaves the window         | `(param)`                       |
| `12`    | Expose          | Window needs redraw (same as expose_hook) | `(param)`                   |
| `33`    | ClientMessage   | Window close button (X button) clicked | `(param)`                       |

**Key difference from convenience hooks:**
- `mlx_key_hook` → fires on KeyRelease (`3`)
- `mlx_hook(win, 2, ...)` → fires on KeyPress, useful for detecting key held down

**Mouse motion (hover) example:**

```python
def on_motion(x, y, param):
    # x, y = current mouse position inside window
    # Use this to implement hover effects
    if BTN_X <= x <= BTN_X + BTN_W and BTN_Y <= y <= BTN_Y + BTN_H:
        state.hovering = True
    else:
        state.hovering = False
    state.needs_redraw = True

mlx.mlx_hook(win_ptr, 6, 0, on_motion, None)
```

> **Note:** Event 6 (MotionNotify) only fires when a **mouse button is held** while moving. If you want mouse position tracking without a button held, poll `mlx_mouse_get_pos` in your loop hook instead.

**Mouse enter/leave window example:**

```python
def on_enter(param):
    state.mouse_in_window = True

def on_leave(param):
    state.mouse_in_window = False

mlx.mlx_hook(win_ptr, 7, 0, on_enter, None)
mlx.mlx_hook(win_ptr, 8, 0, on_leave, None)
```

---

## 15. Mouse Utilities

### `mlx_mouse_hide(mlx_ptr)`

```python
mlx.mlx_mouse_hide(mlx_ptr)
```

Hides the system mouse cursor. The cursor becomes invisible inside the window. Useful for games where you draw your own custom cursor as an image.

- **Returns:** `0` on success

---

### `mlx_mouse_show(mlx_ptr)`

```python
mlx.mlx_mouse_show(mlx_ptr)
```

Makes the system mouse cursor visible again after `mlx_mouse_hide`.

- **Returns:** `0` on success

---

### `mlx_mouse_move(mlx_ptr, x, y)`

```python
mlx.mlx_mouse_move(mlx_ptr, 400, 300)
```

Programmatically moves the mouse cursor to position `(x, y)` inside the window. Useful to snap the cursor to a specific location.

- **Returns:** `0` on success

---

### `mlx_mouse_get_pos(mlx_ptr)` — returns tuple

```python
ret, x, y = mlx.mlx_mouse_get_pos(mlx_ptr)
```

Returns the **current mouse cursor position** without waiting for a click event. Useful for hover effects when polled from the loop hook.

- **Returns:** tuple `(ret, x, y)`
  - `x`, `y`: current cursor position in the window
  - `ret`: `0` on success

**Hover effect using poll (no button needed):**

```python
def on_loop(param):
    _, x, y = mlx.mlx_mouse_get_pos(mlx_ptr)
    hovering = BTN_X <= x <= BTN_X + BTN_W and BTN_Y <= y <= BTN_Y + BTN_H
    if hovering != state.last_hover:
        state.last_hover = hovering
        state.needs_redraw = True
    if state.needs_redraw:
        redraw(hovering)
        state.needs_redraw = False
```

---

## 16. Keyboard Utilities

### `mlx_do_key_autorepeatoff(mlx_ptr)`

```python
mlx.mlx_do_key_autorepeatoff(mlx_ptr)
```

Turns off keyboard auto-repeat. By default, holding a key generates repeated KeyPress events. After calling this, holding a key fires the event only **once** until the key is released.

Useful for games where you want to detect a single press, not continuous rapid-fire.

- **Returns:** `0` on success

---

### `mlx_do_key_autorepeaton(mlx_ptr)`

```python
mlx.mlx_do_key_autorepeaton(mlx_ptr)
```

Turns keyboard auto-repeat back on (default behaviour). Multiple events per second while key is held.

- **Returns:** `0` on success

---

## 17. Screen Info

### `mlx_get_screen_size(mlx_ptr)` — returns tuple

```python
ret, screen_width, screen_height = mlx.mlx_get_screen_size(mlx_ptr)
```

Returns the size of the **physical screen** (monitor). Can be called before creating any window.

- **Returns:** tuple `(ret, width, height)` in pixels
- Useful to center a window or make it fullscreen

**Center a window on screen:**

```python
ret, sw, sh = mlx.mlx_get_screen_size(mlx_ptr)
win_w, win_h = 800, 600
# MLX does not support setting window position, but you know the target offset:
# offset_x = (sw - win_w) // 2
# offset_y = (sh - win_h) // 2
```

> Note: MLX does not provide a function to set the window's position on screen. The OS places it. You can only use the screen size for logic decisions (e.g. choosing tile size to fill the screen).

---

## 18. Sync & Flush

These functions control when pixel data is actually sent to the screen. Normally you don't need them — `mlx_loop` handles flushing automatically.

### `mlx_do_sync(mlx_ptr)`

```python
mlx.mlx_do_sync(mlx_ptr)
```

Flushes all pending drawing commands to the display subsystem. Does **not** wait for the GPU to finish — just makes sure commands are sent, not cached on the software side.

- **Returns:** `0` on success

---

### `mlx_sync(mlx_ptr, cmd, img_or_win_ptr)`

```python
mlx.mlx_sync(mlx_ptr, Mlx.SYNC_IMAGE_WRITABLE, img_ptr)
mlx.mlx_sync(mlx_ptr, Mlx.SYNC_WIN_FLUSH, win_ptr)
mlx.mlx_sync(mlx_ptr, Mlx.SYNC_WIN_COMPLETED, win_ptr)
```

Fine-grained synchronisation control.

**Commands (constants on the `Mlx` class):**

| Constant                   | Value | Meaning                                                         |
|----------------------------|-------|-----------------------------------------------------------------|
| `Mlx.SYNC_IMAGE_WRITABLE`  | `1`   | Wait until image data buffer can be safely written again        |
| `Mlx.SYNC_WIN_FLUSH`       | `2`   | Send all pending requests for this window to the server         |
| `Mlx.SYNC_WIN_COMPLETED`   | `3`   | Flush AND wait for the server to confirm completion             |

**When to use:**
- `SYNC_IMAGE_WRITABLE`: before writing to an image that was just rendered — avoids writing while the GPU is still reading
- `SYNC_WIN_FLUSH`: when you need to force a frame to appear immediately
- `SYNC_WIN_COMPLETED`: when you need to guarantee the frame is fully displayed before continuing

---

## 19. Color System

Colors in MLX are `unsigned int` values. The byte layout depends on the **image format** reported by `mlx_get_data_addr`, which can be either:

- **Format 0:** `B8 G8 R8 A8` (Blue, Green, Red, Alpha)
- **Format 1:** `A8 R8 G8 B8` (Alpha, Red, Green, Blue)

For `mlx_pixel_put` and `mlx_string_put`, use the `0xAARRGGBB` notation:

```
0xFF FF FF FF
  ↑  ↑  ↑  ↑
  A  R  G  B
```

**Common colors:**

```python
WHITE   = 0xFFFFFFFF
BLACK   = 0xFF000000
RED     = 0xFFFF0000
GREEN   = 0xFF00FF00
BLUE    = 0xFF0000FF
YELLOW  = 0xFFFFFF00
CYAN    = 0xFF00FFFF
MAGENTA = 0xFFFF00FF
GREY    = 0xFF888888
```

**For image pixel buffers (raw bytes):**

The 4 bytes are written directly in memory order. On little-endian (all modern PCs):
```python
# Format 0 (B G R A):
color_bytes = bytes([blue, green, red, alpha])

# Format 1 (A R G B):
color_bytes = bytes([alpha, red, green, blue])
```

**Building a color integer from RGB components:**

```python
def rgb(r, g, b, a=255):
    return (a << 24) | (r << 16) | (g << 8) | b
```

**Alpha (transparency):**
- `255` (`0xFF`) = fully opaque
- `0` = fully transparent
- MLX composites images with transparency correctly when rendering

---

## 20. Practical Recipes

### A. Hover Effect on a Button

Detect mouse position each frame, switch to a highlight image when hovering:

```python
BTN_X, BTN_Y, BTN_W, BTN_H = 100, 200, 240, 55

def on_loop(param):
    _, mx, my = mlx.mlx_mouse_get_pos(mlx_ptr)
    hovering = (BTN_X <= mx <= BTN_X + BTN_W and
                BTN_Y <= my <= BTN_Y + BTN_H)
    
    img = img_btn_hover if hovering else img_btn_normal
    mlx.mlx_put_image_to_window(mlx_ptr, win_ptr, img_bg, 0, 0)
    mlx.mlx_put_image_to_window(mlx_ptr, win_ptr, img, BTN_X, BTN_Y)

mlx.mlx_loop_hook(mlx_ptr, on_loop, None)
```

### B. Clickable Image Button

```python
def on_mouse(button, x, y, param):
    if button == 1:  # left click
        if BTN_X <= x <= BTN_X + BTN_W and BTN_Y <= y <= BTN_Y + BTN_H:
            do_action()

mlx.mlx_mouse_hook(win_ptr, on_mouse, None)
```

### C. Custom Mouse Cursor

Hide the system cursor and draw your own image at the mouse position:

```python
mlx.mlx_mouse_hide(mlx_ptr)

def on_loop(param):
    _, mx, my = mlx.mlx_mouse_get_pos(mlx_ptr)
    mlx.mlx_clear_window(mlx_ptr, win_ptr)
    mlx.mlx_put_image_to_window(mlx_ptr, win_ptr, img_bg, 0, 0)
    mlx.mlx_put_image_to_window(mlx_ptr, win_ptr, img_cursor, mx - 8, my - 8)
```

### D. Drawing to an Image Buffer (Fast)

```python
img_ptr = mlx.mlx_new_image(mlx_ptr, 800, 600)
data, bpp, sl, fmt = mlx.mlx_get_data_addr(img_ptr)

# Draw a red diagonal line
for i in range(400):
    offset = i * sl + i * 4
    data[offset:offset + 4] = bytes([0x00, 0x00, 0xFF, 0xFF])  # BGRA red

mlx.mlx_put_image_to_window(mlx_ptr, win_ptr, img_ptr, 0, 0)
```

### E. Multiple Windows

```python
win1 = mlx.mlx_new_window(mlx_ptr, 800, 600, "Window 1")
win2 = mlx.mlx_new_window(mlx_ptr, 400, 300, "Window 2")

# Each window gets its own hooks
mlx.mlx_key_hook(win1, on_key_win1, None)
mlx.mlx_key_hook(win2, on_key_win2, None)
mlx.mlx_mouse_hook(win1, on_mouse_win1, None)
mlx.mlx_mouse_hook(win2, on_mouse_win2, None)

# Close just one window
def on_close_win2(param):
    mlx.mlx_destroy_window(mlx_ptr, win2)

mlx.mlx_hook(win2, 33, 0, on_close_win2, None)
```

### F. Detecting Key Held Down (KeyPress, not KeyRelease)

```python
# KeyPress fires immediately and repeatedly while held
def on_keypress(key, param):
    if key == 65361:  # left arrow
        state.player_x -= 5
        state.needs_redraw = True

mlx.mlx_hook(win_ptr, 2, 1, on_keypress, None)  # event 2 = KeyPress
```

### G. Scene / State Machine

```python
# Switch between "menu" and "game" without opening a new window
state.scene = "menu"

def on_loop(param):
    if not state.needs_redraw:
        return
    mlx.mlx_clear_window(mlx_ptr, win_ptr)
    if state.scene == "menu":
        draw_menu()
    elif state.scene == "game":
        draw_game()
    state.needs_redraw = False

def on_key(key, param):
    if state.scene == "menu" and key == 49:  # '1'
        state.scene = "game"
        state.needs_redraw = True
```

### H. Animation / Frame Counter

```python
import time

state.frame = 0

def on_loop(param):
    state.frame += 1
    # Animate every 10 frames
    frame_img = anim_frames[state.frame % len(anim_frames)]
    mlx.mlx_put_image_to_window(mlx_ptr, win_ptr, frame_img, 0, 0)
```

### I. Clean Program Exit

```python
def shutdown():
    mlx.mlx_destroy_image(mlx_ptr, img_ptr)
    mlx.mlx_destroy_window(mlx_ptr, win_ptr)
    mlx.mlx_release(mlx_ptr)

def on_key(key, param):
    if key == 65307:  # ESC
        mlx.mlx_loop_exit(mlx_ptr)

# After mlx_loop returns:
mlx.mlx_loop(mlx_ptr)
shutdown()
```

---

## 21. Limitations & Known Issues

| Limitation | Detail |
|---|---|
| No window positioning | Cannot set where on screen the window appears |
| Font is fixed | `mlx_string_put` uses one built-in bitmap font, cannot change size or face |
| PNG loader unreliable | Some PNG files fail to load; convert to XPM with ImageMagick for production |
| No Ctrl+C in loop | Python cannot catch SIGINT inside `mlx_loop`. Use `Ctrl+\` or `os._exit(0)` |
| Expose event may not fire | On modern composited desktops the expose hook is unreliable; use `loop_hook` + dirty flag |
| MotionNotify requires button held | Event 6 only fires during click-drag; use `mlx_mouse_get_pos` polling for hover |
| Alpha in `mlx_string_put` | Alpha channel of the color is always forced to 255 (opaque) — you cannot make text semi-transparent |
| `mlx_pixel_put` is slow | Each call is a separate GPU command; use image buffers for bulk drawing |
| No text input | There is no text field or input widget; you must build your own using key hooks |
| No built-in UI | No buttons, checkboxes, sliders — all must be built using images and hit-testing |
| Image format endian | `mlx_get_data_addr` format can be `0` or `1` — always check before writing pixel bytes |

---

## 22. Quick Reference Card

```
INIT
  mlx_ptr = mlx.mlx_init()
  mlx.mlx_release(mlx_ptr)

WINDOW
  win_ptr = mlx.mlx_new_window(mlx_ptr, w, h, "title")
  mlx.mlx_clear_window(mlx_ptr, win_ptr)       # fill black
  mlx.mlx_destroy_window(mlx_ptr, win_ptr)

DRAW DIRECT (slow)
  mlx.mlx_pixel_put(mlx_ptr, win_ptr, x, y, color)
  mlx.mlx_string_put(mlx_ptr, win_ptr, x, y, color, "text")

IMAGES (fast)
  img = mlx.mlx_new_image(mlx_ptr, w, h)
  data, bpp, sl, fmt = mlx.mlx_get_data_addr(img)
  mlx.mlx_put_image_to_window(mlx_ptr, win_ptr, img, x, y)
  mlx.mlx_destroy_image(mlx_ptr, img)

LOAD FILES
  img, w, h = mlx.mlx_xpm_file_to_image(mlx_ptr, "file.xpm")  # recommended
  img, w, h = mlx.mlx_png_file_to_image(mlx_ptr, "file.png")  # unreliable

LOOP & EVENTS
  mlx.mlx_loop(mlx_ptr)                         # starts loop, never returns
  mlx.mlx_loop_exit(mlx_ptr)                    # exit from inside a hook
  mlx.mlx_key_hook(win_ptr, fn(key,p), param)   # key released
  mlx.mlx_mouse_hook(win_ptr, fn(btn,x,y,p), p) # mouse click
  mlx.mlx_expose_hook(win_ptr, fn(p), param)     # window needs redraw
  mlx.mlx_loop_hook(mlx_ptr, fn(p), param)       # every frame

GENERIC HOOK (all events)
  mlx.mlx_hook(win_ptr, x_event, x_mask, fn, param)
  # x_event: 2=KeyPress  3=KeyRelease  4=BtnPress  5=BtnRelease
  #           6=Motion    7=Enter       8=Leave     33=WindowClose

MOUSE UTILS
  mlx.mlx_mouse_hide(mlx_ptr)
  mlx.mlx_mouse_show(mlx_ptr)
  mlx.mlx_mouse_move(mlx_ptr, x, y)
  ret, x, y = mlx.mlx_mouse_get_pos(mlx_ptr)

KEYBOARD UTILS
  mlx.mlx_do_key_autorepeatoff(mlx_ptr)
  mlx.mlx_do_key_autorepeaton(mlx_ptr)

SCREEN
  ret, w, h = mlx.mlx_get_screen_size(mlx_ptr)

SYNC
  mlx.mlx_do_sync(mlx_ptr)
  mlx.mlx_sync(mlx_ptr, Mlx.SYNC_IMAGE_WRITABLE, img)
  mlx.mlx_sync(mlx_ptr, Mlx.SYNC_WIN_FLUSH, win)
  mlx.mlx_sync(mlx_ptr, Mlx.SYNC_WIN_COMPLETED, win)

COLORS  0xAARRGGBB
  WHITE=0xFFFFFFFF  BLACK=0xFF000000  RED=0xFFFF0000
  GREEN=0xFF00FF00  BLUE=0xFF0000FF   YELLOW=0xFFFFFF00

KEY CODES (Linux)
  ESC=65307  Enter=65293  Space=32
  Up=65362   Down=65364   Left=65361  Right=65363
  'a'=97     'z'=122      '0'=48      '1'=49
```

---

*Documentation written from source: `mlx.py`, `mlx.h`, man pages, C source files, and test examples — mlx_CLXV v2.2*
