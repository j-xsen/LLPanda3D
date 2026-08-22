# DisplayInformation

**Source:** `panda/src/display/displayInformation.h` (+ `.cxx`, no `.I`)
**Inherits:** (none — standalone data class) **Inherited by:** (none)

A queried snapshot of hardware/OS display and system capabilities: available
display modes, GPU vendor/device/driver identification, shader model,
video/texture memory, host system memory and paging stats, CPU
identification/frequency/core count, and OS version. Obtained via
`GraphicsPipe::get_display_information()` — see
[GraphicsPipe.md](GraphicsPipe.md), which this doc does not re-explain.
`DisplaySearchParameters` (folded in below) is a small companion class used
to constrain a display-mode search (e.g.
`GraphicsPipe::find_all_display_modes()`-style queries).

## Behavior notes

- **All data members are `public`, not `private`** — unusually for a Panda3D
  class, `DisplayInformation` is a plain struct with a
  `PUBLISHED` accessor layer bolted on for the scripting interface; C++
  code populating or reading it (typically a platform-specific
  `GraphicsPipe` subclass) commonly touches the `_foo` fields directly
  rather than going through `get_foo()`.
- **Most fields are populated only on platforms/backends that bother to
  fill them in** (historically Windows/DirectX, via WMI and DXGI/DirectX
  queries) — the default constructor zeroes or `-1`-fills every field, sets
  `_state` to `DS_unknown`, and `_shader_model` to `SM_00`. A
  `GraphicsPipe` that doesn't implement hardware detection for its platform
  never overwrites most of these, so most getters read as
  `0`/`-1`/`DS_unknown` outside of that DirectX/Windows path. Always check
  `get_display_state()` for `DS_success` before trusting the rest of the
  structure.
- **Memory and CPU-frequency stats are refreshed on demand via function
  pointers, not automatically.** `_get_memory_information_function` and
  `_update_cpu_frequency_function` are raw C function pointers
  (`void(*)(DisplayInformation*)` and `int(*)(int, DisplayInformation*)`
  respectively) that a platform backend installs into the instance; the
  public `update_memory_information()` and
  `update_cpu_frequency(processor_number)` methods just no-op if the
  corresponding pointer is `nullptr`, otherwise invoke it (which mutates
  `this`'s memory/frequency fields in place). Nothing calls these
  automatically per-frame — an application must call them explicitly to
  get current numbers.
- **`get_cpu_time()` is a static, direct `rdtsc` wrapper** on x86/x86-64
  (`__rdtsc()` intrinsic under MSVC/GCC, inline `rdtsc` asm otherwise) —
  it returns the raw CPU timestamp counter, not a wall-clock time; on
  non-x86 architectures it unconditionally returns `0`.
- **`get_display_mode(display_index)` asserts the index is in range in
  debug builds** (`nassertr`, returning a static zeroed `DisplayMode` on
  failure) but the parallel "older interface" methods
  (`get_display_mode_width/height/bits_per_pixel/refresh_rate/fullscreen_only`)
  instead silently return `0` for an out-of-range index — two different
  failure conventions for what's otherwise the same data, so don't assume
  one implies the other.
- **`DisplayMode::fullscreen_only`/`bits_per_pixel`/`refresh_rate` are `int`,
  not `bool`**, and `DisplayMode::output()` only prints
  `bits_per_pixel`/`refresh_rate`/the "(fullscreen only)" suffix when the
  respective field is `> 0` — a `0` value means "not reported/not
  applicable," not "false."

## DisplaySearchParameters

A tiny public-data-member helper (no methods beyond setters — same public-field style as `DisplayInformation`) used to bound a display-mode search: minimum/maximum width, height, and bits-per-pixel. Its constructor seeds sane broad defaults rather than zeros:

```cpp
_minimum_width = 640;   _maximum_width  = 6400;
_minimum_height = 480;  _maximum_height = 4800;
_minimum_bits_per_pixel = 16; _maximum_bits_per_pixel = 32;
```

| Signature |
|---|
| `DisplaySearchParameters()` — seeds the defaults above |
| `void set_minimum_width(int)` / `set_maximum_width(int)` |
| `void set_minimum_height(int)` / `set_maximum_height(int)` |
| `void set_minimum_bits_per_pixel(int)` / `set_maximum_bits_per_pixel(int)` |

## API

### DisplayMode (struct)

| Field | Type |
|---|---|
| `width`, `height`, `bits_per_pixel`, `refresh_rate`, `fullscreen_only` | `int` |

Plus `operator==`/`operator!=` (exact field comparison) and `void output(std::ostream&) const`.

### DisplayInformation — status / display modes

| Signature | Notes |
|---|---|
| `DisplayInformation()` / `~DisplayInformation()` | Frees `_display_mode_array` if allocated. |
| `int get_display_state()` | One of the `DetectionState` enum values below. |
| `int get_maximum_window_width()` / `get_maximum_window_height()` / `get_window_bits_per_pixel()` | |
| `int get_total_display_modes()` | |
| `const DisplayMode &get_display_mode(int display_index)` | Asserts in-range in debug builds. |
| `MAKE_SEQ(get_display_modes, ...)` | Sequence accessor over all display modes (scripting-interface convenience). |
| `int get_current_display_mode_index() const` | `-1` if undetermined. |
| `int get_display_mode_width/height/bits_per_pixel/refresh_rate/fullscreen_only(int display_index)` | Older per-field interface; `0` on out-of-range instead of asserting. |

`enum DetectionState { DS_unknown, DS_success, DS_direct_3d_create_error, DS_create_window_error, DS_create_device_error }`

### DisplayInformation — GPU / driver

| Signature |
|---|
| `GraphicsStateGuardian::ShaderModel get_shader_model()` |
| `int get_video_memory()` / `int get_texture_memory()` |
| `int get_vendor_id()` / `int get_device_id()` |
| `int get_driver_product()` / `get_driver_version()` / `get_driver_sub_version()` / `get_driver_build()` |
| `int get_driver_date_month()` / `get_driver_date_day()` / `get_driver_date_year()` |

### DisplayInformation — system memory (call `update_memory_information()` first)

| Signature |
|---|
| `void update_memory_information()` |
| `uint64_t get_physical_memory()` / `get_available_physical_memory()` |
| `uint64_t get_page_file_size()` / `get_available_page_file_size()` |
| `uint64_t get_process_virtual_memory()` / `get_available_process_virtual_memory()` |
| `int get_memory_load()` |
| `uint64_t get_page_fault_count()` / `get_process_memory()` / `get_peak_process_memory()` |
| `uint64_t get_page_file_usage()` / `get_peak_page_file_usage()` |

### DisplayInformation — CPU / OS

| Signature | Notes |
|---|---|
| `const std::string &get_cpu_vendor_string() const` / `get_cpu_brand_string() const` | |
| `unsigned int get_cpu_version_information()` / `get_cpu_brand_index()` | |
| `uint64_t get_cpu_frequency()` | |
| `static uint64_t get_cpu_time()` | Raw `rdtsc`; `0` on non-x86. |
| `uint64_t get_maximum_cpu_frequency()` / `get_current_cpu_frequency()` | |
| `void update_cpu_frequency(int processor_number)` | See behavior notes on the function-pointer indirection. |
| `int get_num_cpu_cores()` / `get_num_logical_cpus()` | |
| `int get_os_version_major()` / `get_os_version_minor()` / `get_os_version_build()` / `get_os_platform_id()` | |

## Usage

```cpp
#include "graphicsPipe.h"
#include "displayInformation.h"

DisplayInformation *info = pipe->get_display_information();
if (info->get_display_state() == DisplayInformation::DS_success) {
  info->update_memory_information();
  nout << "Video memory: " << info->get_video_memory() << " bytes\n";
  nout << "Available physical memory: "
       << info->get_available_physical_memory() << " bytes\n";

  for (int i = 0; i < info->get_total_display_modes(); ++i) {
    nout << info->get_display_mode(i) << "\n";
  }
}
```

## See also

- [GraphicsPipe.md](GraphicsPipe.md) — `get_display_information()` is the accessor for this class; `GraphicsDevice` (folded there) is the related vendor/adapter handle.
- [GraphicsStateGuardian.md](GraphicsStateGuardian.md) — `ShaderModel` enum returned by `get_shader_model()`.
