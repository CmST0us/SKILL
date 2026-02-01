# UVC Gadget Skill

This skill provides knowledge about configuring and testing USB Video Class (UVC) gadget on Rockchip platforms.


## UVC Gadget Configuration Script

### Script
#### Rockchip platform
```bash
#!/bin/bash
#
# UVC Gadget Configuration Script
# Configures USB Video Class gadget on Rockchip platform
#

set -e

# ============================================================================
# Configuration
# ============================================================================

GADGET_NAME="rockchip"
GADGET_PATH="/sys/kernel/config/usb_gadget/${GADGET_NAME}"
UVC_FUNC="uvc.gs6"

# USB Device Descriptor
USB_VID="0x2207"        # Rockchip
USB_PID="0x0005"        # UVC Camera
USB_SERIAL="2020"
USB_MANUFACTURER="Rockchip"
USB_PRODUCT="UVC"

# ============================================================================
# Resolution Definitions
# Add new resolutions here using the format:
#   "NAME:WIDTH:HEIGHT:INTERVALS:FORMAT"
#
# INTERVALS: Frame intervals in 100ns units, comma separated
#   333333  = 30fps
#   166666  = 60fps
#   666666  = 15fps
#   1000000 = 10fps
#
# FORMAT: nv12 or mjpeg
# ============================================================================

RESOLUTIONS=(
    # YUY2 (Uncompressed) formats - 1080p
    "1080p_30_yuy2:1920:1080:333333:yuy2"      # 30fps
    "1080p_60_yuy2:1920:1080:166666:yuy2"      # 60fps

    # MJPEG formats - 1080p
    "1080p_30_mjpeg:1920:1080:333333:mjpeg"    # 30fps
    "1080p_60_mjpeg:1920:1080:166666:mjpeg"    # 60fps

    # Uncomment to add more resolutions:
    # "720p_30_yuy2:1280:720:333333:yuy2"
    # "720p_60_yuy2:1280:720:166666:yuy2"
    # "480p_30_yuy2:640:480:333333:yuy2"
    # "4k_30_mjpeg:3840:2160:333333:mjpeg"
)

# ============================================================================
# Helper Functions
# ============================================================================

log_info() {
    echo -e "\033[34m[INFO]\033[0m $1"
}

log_error() {
    echo -e "\033[31m[ERROR]\033[0m $1" >&2
}

log_success() {
    echo -e "\033[32m[OK]\033[0m $1"
}

log_warn() {
    echo -e "\033[33m[WARN]\033[0m $1"
}

# Get UDC name
get_udc() {
    ls /sys/class/udc/ 2>/dev/null | head -1
}

# Convert intervals to fps string for display
intervals_to_fps() {
    local intervals=$1
    local fps_list=""

    IFS=',' read -ra INTS <<< "$intervals"
    for int in "${INTS[@]}"; do
        local fps=$((10000000 / int))
        if [ -n "$fps_list" ]; then
            fps_list="${fps_list}/${fps}"
        else
            fps_list="${fps}"
        fi
    done
    echo "${fps_list}fps"
}

# ============================================================================
# Gadget Configuration Functions
# ============================================================================

check_configfs() {
    if ! mountpoint -q /sys/kernel/config 2>/dev/null; then
        log_info "Mounting configfs..."
        mount -t configfs none /sys/kernel/config
    fi
}

create_gadget() {
    log_info "Creating USB gadget..."

    mkdir -p "${GADGET_PATH}"
    mkdir -p "${GADGET_PATH}/strings/0x409"
    mkdir -p "${GADGET_PATH}/configs/b.1/strings/0x409"

    # Set USB device descriptor
    echo "${USB_VID}" > "${GADGET_PATH}/idVendor"
    echo "${USB_PID}" > "${GADGET_PATH}/idProduct"
    echo 0x0310 > "${GADGET_PATH}/bcdDevice"
    echo 0x0200 > "${GADGET_PATH}/bcdUSB"

    # Create English strings
    echo "${USB_SERIAL}" > "${GADGET_PATH}/strings/0x409/serialnumber"
    echo "${USB_MANUFACTURER}" > "${GADGET_PATH}/strings/0x409/manufacturer"
    echo "${USB_PRODUCT}" > "${GADGET_PATH}/strings/0x409/product"

    # OS descriptor for Windows
    echo 0x1 > "${GADGET_PATH}/os_desc/b_vendor_code"
    echo "MSFT100" > "${GADGET_PATH}/os_desc/qw_sign"

    # Config
    echo 500 > "${GADGET_PATH}/configs/b.1/MaxPower"
    echo "uvc" > "${GADGET_PATH}/configs/b.1/strings/0x409/configuration"
    ln -sf "${GADGET_PATH}/configs/b.1" "${GADGET_PATH}/os_desc/b.1" 2>/dev/null || true

    log_success "Gadget created"
}

create_uvc_function() {
    log_info "Creating UVC function..."

    local func_path="${GADGET_PATH}/functions/${UVC_FUNC}"
    mkdir -p "${func_path}"

    # Create control interface
    mkdir -p "${func_path}/control/header/h"
    ln -sf "${func_path}/control/header/h" "${func_path}/control/class/fs/h" 2>/dev/null || true
    ln -sf "${func_path}/control/header/h" "${func_path}/control/class/ss/h" 2>/dev/null || true

    # Create streaming header
    mkdir -p "${func_path}/streaming/header/h"

    # Create format directories
    mkdir -p "${func_path}/streaming/uncompressed/u"
    mkdir -p "${func_path}/streaming/mjpeg/m"

    log_success "UVC function created"
}

# Add uncompressed format (NV12/YUYV)
add_uncompressed_format() {
    local name=$1
    local width=$2
    local height=$3
    local intervals=$4

    local func_path="${GADGET_PATH}/functions/${UVC_FUNC}"
    local format_path="${func_path}/streaming/uncompressed/u"
    local frame_path="${format_path}/${name}"

    local fps_str=$(intervals_to_fps "$intervals")
    log_info "Adding YUY2: ${width}x${height}@${fps_str}"

    mkdir -p "${frame_path}"

    # Get first interval as default
    local default_interval=$(echo "$intervals" | cut -d',' -f1)

    # Calculate sizes
    local frame_size=$((width * height * 2))  # YUY2: 2 bytes per pixel
    local bitrate=$((width * height * 16 * 10000000 / default_interval))  # 16 bits per pixel

    echo "$width" > "${frame_path}/wWidth"
    echo "$height" > "${frame_path}/wHeight"
    echo "$bitrate" > "${frame_path}/dwMinBitRate"
    echo "$bitrate" > "${frame_path}/dwMaxBitRate"
    echo "$frame_size" > "${frame_path}/dwMaxVideoFrameBufferSize"
    echo "$default_interval" > "${frame_path}/dwDefaultFrameInterval"

    # Write frame interval
    echo "$default_interval" > "${frame_path}/dwFrameInterval"

    log_success "Added ${name}"
}

# Add MJPEG format
add_mjpeg_format() {
    local name=$1
    local width=$2
    local height=$3
    local intervals=$4

    local func_path="${GADGET_PATH}/functions/${UVC_FUNC}"
    local format_path="${func_path}/streaming/mjpeg/m"
    local frame_path="${format_path}/${name}"

    local fps_str=$(intervals_to_fps "$intervals")
    log_info "Adding MJPEG: ${width}x${height}@${fps_str}"

    mkdir -p "${frame_path}"

    local default_interval=$(echo "$intervals" | cut -d',' -f1)

    # MJPEG compressed, estimate sizes
    local frame_size=$((width * height / 2))
    local bitrate=$((width * height * 4 * 10000000 / default_interval))

    echo "$width" > "${frame_path}/wWidth"
    echo "$height" > "${frame_path}/wHeight"
    echo "$bitrate" > "${frame_path}/dwMinBitRate"
    echo "$((bitrate * 2))" > "${frame_path}/dwMaxBitRate"
    echo "$frame_size" > "${frame_path}/dwMaxVideoFrameBufferSize"
    echo "$default_interval" > "${frame_path}/dwDefaultFrameInterval"
    echo "$default_interval" > "${frame_path}/dwFrameInterval"

    log_success "Added ${name}"
}

# Parse and add resolution from definition string
add_resolution() {
    local def=$1

    IFS=':' read -r name width height intervals format <<< "$def"

    case "$format" in
        yuy2|YUY2|yuyv|YUYV|uncompressed)
            add_uncompressed_format "$name" "$width" "$height" "$intervals"
            ;;
        mjpeg|MJPEG|jpeg|JPEG)
            add_mjpeg_format "$name" "$width" "$height" "$intervals"
            ;;
        *)
            log_error "Unknown format: $format"
            return 1
            ;;
    esac
}

# Configure all resolutions
configure_resolutions() {
    log_info "Configuring resolutions..."

    for res in "${RESOLUTIONS[@]}"; do
        add_resolution "$res"
    done

    log_success "All resolutions configured"
}

# Link streaming classes
link_streaming_classes() {
    local func_path="${GADGET_PATH}/functions/${UVC_FUNC}"

    log_info "Linking streaming classes..."

    # Link format directories to header
    ln -sf "${func_path}/streaming/uncompressed/u" "${func_path}/streaming/header/h/u" 2>/dev/null || true
    ln -sf "${func_path}/streaming/mjpeg/m" "${func_path}/streaming/header/h/m" 2>/dev/null || true

    # Link header to classes
    ln -sf "${func_path}/streaming/header/h" "${func_path}/streaming/class/fs/h" 2>/dev/null || true
    ln -sf "${func_path}/streaming/header/h" "${func_path}/streaming/class/hs/h" 2>/dev/null || true
    ln -sf "${func_path}/streaming/header/h" "${func_path}/streaming/class/ss/h" 2>/dev/null || true

    log_success "Streaming classes linked"
}

# Link function to configuration
link_function() {
    log_info "Linking function to configuration..."

    local config_path="${GADGET_PATH}/configs/b.1"

    # Remove old links
    for f in "${config_path}"/f[0-9]*; do
        [ -L "$f" ] && rm -f "$f"
    done
    rm -f "${config_path}/ffs.adb" 2>/dev/null || true

    # Link UVC function
    ln -sf "${GADGET_PATH}/functions/${UVC_FUNC}" "${config_path}/f1"

    log_success "Function linked"
}

# Enable gadget
enable_gadget() {
    local udc=$(get_udc)

    if [ -z "$udc" ]; then
        log_error "No UDC found!"
        return 1
    fi

    log_info "Enabling gadget on ${udc}..."
    echo "${udc}" > "${GADGET_PATH}/UDC"

    log_success "Gadget enabled"
}

# Disable gadget
disable_gadget() {
    if [ -f "${GADGET_PATH}/UDC" ]; then
        local current=$(cat "${GADGET_PATH}/UDC" 2>/dev/null)
        if [ -n "$current" ]; then
            log_info "Disabling gadget..."
            echo "" > "${GADGET_PATH}/UDC" 2>/dev/null || true
            log_success "Gadget disabled"
        fi
    fi
}

# Remove gadget completely
remove_gadget() {
    log_info "Removing gadget..."

    disable_gadget

    local func_path="${GADGET_PATH}/functions/${UVC_FUNC}"

    # Remove config links
    rm -f "${GADGET_PATH}/configs/b.1/f"* 2>/dev/null || true
    rm -f "${GADGET_PATH}/os_desc/b.1" 2>/dev/null || true

    # Remove streaming class links
    rm -f "${func_path}/streaming/class/fs/h" 2>/dev/null || true
    rm -f "${func_path}/streaming/class/hs/h" 2>/dev/null || true
    rm -f "${func_path}/streaming/class/ss/h" 2>/dev/null || true

    # Remove control class links
    rm -f "${func_path}/control/class/fs/h" 2>/dev/null || true
    rm -f "${func_path}/control/class/ss/h" 2>/dev/null || true

    # Remove header links
    rm -f "${func_path}/streaming/header/h/u" 2>/dev/null || true
    rm -f "${func_path}/streaming/header/h/m" 2>/dev/null || true
    rm -f "${func_path}/streaming/header/h/f" 2>/dev/null || true

    # Remove frame directories
    for dir in "${func_path}/streaming/uncompressed/u"/*; do
        [ -d "$dir" ] && rmdir "$dir" 2>/dev/null || true
    done
    for dir in "${func_path}/streaming/mjpeg/m"/*; do
        [ -d "$dir" ] && rmdir "$dir" 2>/dev/null || true
    done

    # Remove format directories
    rmdir "${func_path}/streaming/uncompressed/u" 2>/dev/null || true
    rmdir "${func_path}/streaming/mjpeg/m" 2>/dev/null || true

    # Remove headers
    rmdir "${func_path}/streaming/header/h" 2>/dev/null || true
    rmdir "${func_path}/control/header/h" 2>/dev/null || true

    # Remove function
    rmdir "${func_path}" 2>/dev/null || true

    # Remove config
    rmdir "${GADGET_PATH}/configs/b.1/strings/0x409" 2>/dev/null || true
    rmdir "${GADGET_PATH}/configs/b.1" 2>/dev/null || true

    # Remove gadget
    rmdir "${GADGET_PATH}/strings/0x409" 2>/dev/null || true
    rmdir "${GADGET_PATH}" 2>/dev/null || true

    log_success "Gadget removed"
}

# ============================================================================
# Status Functions
# ============================================================================

show_status() {
    echo ""
    echo "========================================"
    echo "       UVC Gadget Status"
    echo "========================================"

    if [ ! -d "${GADGET_PATH}" ]; then
        echo "Status: NOT CONFIGURED"
        echo "========================================"
        return
    fi

    local udc_content=$(cat "${GADGET_PATH}/UDC" 2>/dev/null)
    if [ -n "$udc_content" ]; then
        echo -e "Status: \033[32mENABLED\033[0m (UDC: ${udc_content})"
    else
        echo -e "Status: \033[33mDISABLED\033[0m"
    fi

    # Find video device
    local video_dev=""
    for v in /sys/class/video4linux/video*; do
        if [ -f "$v/name" ]; then
            local name=$(cat "$v/name" 2>/dev/null)
            if echo "$name" | grep -qi "gadget\|uvc\|dwc3"; then
                video_dev=$(basename "$v")
                break
            fi
        fi
    done

    if [ -n "$video_dev" ]; then
        echo "Video Device: /dev/${video_dev}"
    fi

    echo ""
    echo "Configured Formats:"

    local func_path="${GADGET_PATH}/functions/${UVC_FUNC}"

    # List uncompressed formats
    if [ -d "${func_path}/streaming/uncompressed/u" ]; then
        local has_frames=false
        for frame in "${func_path}/streaming/uncompressed/u"/*; do
            if [ -d "$frame" ]; then
                has_frames=true
                break
            fi
        done

        if [ "$has_frames" = true ]; then
            echo "  YUY2 (Uncompressed):"
            for frame in "${func_path}/streaming/uncompressed/u"/*; do
                if [ -d "$frame" ]; then
                    local name=$(basename "$frame")
                    local w=$(cat "$frame/wWidth" 2>/dev/null)
                    local h=$(cat "$frame/wHeight" 2>/dev/null)
                    local intervals=$(cat "$frame/dwFrameInterval" 2>/dev/null | tr '\n' ',' | sed 's/,$//')
                    local fps_str=$(intervals_to_fps "$intervals")
                    echo "    - ${name}: ${w}x${h}@${fps_str}"
                fi
            done
        fi
    fi

    # List MJPEG formats
    if [ -d "${func_path}/streaming/mjpeg/m" ]; then
        local has_frames=false
        for frame in "${func_path}/streaming/mjpeg/m"/*; do
            if [ -d "$frame" ]; then
                has_frames=true
                break
            fi
        done

        if [ "$has_frames" = true ]; then
            echo "  MJPEG:"
            for frame in "${func_path}/streaming/mjpeg/m"/*; do
                if [ -d "$frame" ]; then
                    local name=$(basename "$frame")
                    local w=$(cat "$frame/wWidth" 2>/dev/null)
                    local h=$(cat "$frame/wHeight" 2>/dev/null)
                    local intervals=$(cat "$frame/dwFrameInterval" 2>/dev/null | tr '\n' ',' | sed 's/,$//')
                    local fps_str=$(intervals_to_fps "$intervals")
                    echo "    - ${name}: ${w}x${h}@${fps_str}"
                fi
            done
        fi
    fi

    echo "========================================"
    echo ""
}

# ============================================================================
# Main Commands
# ============================================================================

do_start() {
    log_info "Starting UVC Gadget..."

    # Stop udev if running (may interfere)
    if [ -x /etc/init.d/S10udev ]; then
        /etc/init.d/S10udev stop 2>/dev/null || true
    fi

    # Check if already running
    if [ -d "${GADGET_PATH}" ]; then
        local udc_content=$(cat "${GADGET_PATH}/UDC" 2>/dev/null)
        if [ -n "$udc_content" ]; then
            log_warn "Gadget already running"
            show_status
            return 0
        fi
        # Configured but not enabled
        log_info "Gadget configured, enabling..."
        enable_gadget
        show_status
        return 0
    fi

    # Full setup
    check_configfs
    create_gadget
    create_uvc_function
    configure_resolutions
    link_streaming_classes
    link_function
    enable_gadget

    show_status
    log_success "UVC Gadget started successfully!"
}

do_stop() {
    log_info "Stopping UVC Gadget..."
    disable_gadget
    log_success "UVC Gadget stopped"
}

do_restart() {
    do_remove
    sleep 1
    do_start
}

do_remove() {
    log_info "Removing UVC Gadget configuration..."
    remove_gadget
    log_success "UVC Gadget removed"
}

show_help() {
    cat << 'EOF'
UVC Gadget Configuration Script

Usage: uvc_config.sh <command> [options]

Commands:
    start       Configure and enable UVC gadget
    stop        Disable UVC gadget (keep configuration)
    restart     Remove and reconfigure UVC gadget
    remove      Remove UVC gadget configuration completely
    status      Show current status and configured formats
    add         Add a new resolution dynamically
    help        Show this help message

Examples:
    uvc_config.sh start
    uvc_config.sh stop
    uvc_config.sh status
    uvc_config.sh add 720p_yuy2 1280 720 333333 yuy2
    uvc_config.sh add 4k_mjpeg 3840 2160 333333 mjpeg

Frame Interval Reference (dwFrameInterval in 100ns units):
    166666  = 60 fps
    333333  = 30 fps
    400000  = 25 fps
    666666  = 15 fps
    1000000 = 10 fps
    2000000 =  5 fps

Adding New Resolutions Permanently:
    Edit the RESOLUTIONS array in this script:

    RESOLUTIONS=(
        "name:width:height:intervals:format"
        ...
    )

    Format: "NAME:WIDTH:HEIGHT:INTERVALS:FORMAT"
    - NAME: Unique identifier (e.g., 1080p_nv12)
    - WIDTH: Frame width in pixels
    - HEIGHT: Frame height in pixels
    - INTERVALS: Comma-separated frame intervals (100ns units)
    - FORMAT: yuy2 or mjpeg

EOF

    echo "Current Configured Resolutions:"
    for res in "${RESOLUTIONS[@]}"; do
        IFS=':' read -r name width height intervals format <<< "$res"
        local fps_str=$(intervals_to_fps "$intervals")
        echo "    ${name}: ${width}x${height}@${fps_str} (${format})"
    done
    echo ""
}

do_add() {
    if [ $# -lt 5 ]; then
        echo "Usage: $0 add <name> <width> <height> <intervals> <format>"
        echo ""
        echo "  name:      Unique name (e.g., 720p_nv12)"
        echo "  width:     Frame width in pixels"
        echo "  height:    Frame height in pixels"
        echo "  intervals: Frame intervals, comma-separated (333333=30fps, 166666=60fps)"
        echo "  format:    yuy2 or mjpeg"
        echo ""
        echo "Examples:"
        echo "  $0 add 720p_30_yuy2 1280 720 333333 yuy2"
        echo "  $0 add 720p_60_yuy2 1280 720 166666 yuy2"
        echo "  $0 add 4k_mjpeg 3840 2160 333333 mjpeg"
        return 1
    fi

    local name=$1
    local width=$2
    local height=$3
    local intervals=$4
    local format=$5

    # Check if gadget exists
    if [ ! -d "${GADGET_PATH}/functions/${UVC_FUNC}" ]; then
        log_error "UVC function not found. Run '$0 start' first."
        return 1
    fi

    # Disable gadget temporarily
    local was_enabled=false
    local udc_content=$(cat "${GADGET_PATH}/UDC" 2>/dev/null)
    if [ -n "$udc_content" ]; then
        was_enabled=true
        disable_gadget
        sleep 2
    fi

    # Add the resolution
    add_resolution "${name}:${width}:${height}:${intervals}:${format}"
    link_streaming_classes

    # Re-enable if it was enabled
    if [ "$was_enabled" = true ]; then
        enable_gadget
    fi

    echo ""
    log_success "Resolution added!"
    echo ""
    echo "To make this permanent, add to RESOLUTIONS array in this script:"
    echo "    \"${name}:${width}:${height}:${intervals}:${format}\""
    echo ""

    show_status
}

# ============================================================================
# Main Entry Point
# ============================================================================

case "${1:-}" in
    start)
        do_start
        ;;
    stop)
        do_stop
        ;;
    restart)
        do_restart
        ;;
    remove|unload)
        do_remove
        ;;
    status)
        show_status
        ;;
    add)
        shift
        do_add "$@"
        ;;
    help|--help|-h)
        show_help
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|remove|status|add|help}"
        echo "Run '$0 help' for more information."
        exit 1
        ;;
esac

```

### Commands

| Command | Description |
|---------|-------------|
| `uvc_config.sh start` | Configure and enable UVC gadget |
| `uvc_config.sh stop` | Disable UVC gadget (keep configuration) |
| `uvc_config.sh restart` | Remove and reconfigure UVC gadget |
| `uvc_config.sh remove` | Remove UVC gadget configuration completely |
| `uvc_config.sh status` | Show current status and configured formats |
| `uvc_config.sh help` | Show help message |

### Supported Formats

| Format | Type | Description |
|--------|------|-------------|
| YUY2 | Uncompressed | Standard UVC uncompressed format (2 bytes/pixel) |
| MJPEG | Compressed | Motion JPEG compressed format |

### Frame Interval Reference (100ns units)

| Interval | FPS |
|----------|-----|
| 166666 | 60 fps |
| 333333 | 30 fps |
| 400000 | 25 fps |
| 666666 | 15 fps |
| 1000000 | 10 fps |

### Adding New Resolutions

Edit the `RESOLUTIONS` array in `uvc_config.sh`:

```bash
RESOLUTIONS=(
    "NAME:WIDTH:HEIGHT:INTERVAL:FORMAT"
    # Examples:
    "1080p_30_yuy2:1920:1080:333333:yuy2"
    "1080p_60_yuy2:1920:1080:166666:yuy2"
    "720p_30_yuy2:1280:720:333333:yuy2"
    "1080p_30_mjpeg:1920:1080:333333:mjpeg"
)
```

## GStreamer Testing

### Prerequisites
- UVC gadget must be configured: `uvc_config.sh start`
- Device should be connected to USB host for stable streaming

### YUY2 (Uncompressed) Test

```bash
# 1080p@30fps
gst-launch-1.0 videotestsrc pattern=smpte ! \
    video/x-raw,format=YUY2,width=1920,height=1080,framerate=30/1 ! \
    uvcsink v4l2sink::device=/dev/video0

# 1080p@60fps
gst-launch-1.0 videotestsrc pattern=smpte ! \
    video/x-raw,format=YUY2,width=1920,height=1080,framerate=60/1 ! \
    uvcsink v4l2sink::device=/dev/video0
```

### MJPEG (Compressed) Test

```bash
# 1080p@30fps
gst-launch-1.0 videotestsrc pattern=smpte ! \
    video/x-raw,width=1920,height=1080,framerate=30/1 ! \
    jpegenc ! \
    image/jpeg,width=1920,height=1080,framerate=30/1 ! \
    uvcsink v4l2sink::device=/dev/video0

# 1080p@60fps
gst-launch-1.0 videotestsrc pattern=smpte ! \
    video/x-raw,width=1920,height=1080,framerate=60/1 ! \
    jpegenc ! \
    image/jpeg,width=1920,height=1080,framerate=60/1 ! \
    uvcsink v4l2sink::device=/dev/video0
```

### Test Patterns

| Pattern | Description |
|---------|-------------|
| `smpte` | SMPTE color bars |
| `snow` | Random noise |
| `ball` | Moving ball |
| `checkers-8` | Checkerboard |

### Stop GStreamer

```bash
pkill gst-launch
```

## ConfigFS Structure

### Rockchip

```
/sys/kernel/config/usb_gadget/rockchip/
├── UDC                          # UDC controller name
├── idVendor                     # 0x2207 (Rockchip)
├── idProduct                    # 0x0005
├── functions/uvc.gs6/
│   ├── streaming/
│   │   ├── uncompressed/u/      # YUY2 frames
│   │   │   └── <frame_name>/
│   │   │       ├── wWidth
│   │   │       ├── wHeight
│   │   │       ├── dwDefaultFrameInterval
│   │   │       └── dwFrameInterval
│   │   ├── mjpeg/m/             # MJPEG frames
│   │   │   └── <frame_name>/
│   │   └── header/h/
│   └── control/
└── configs/b.1/
```

## Important Notes

1. **Format**: UVC uncompressed format is **YUY2** (not NV12). If input is NV12, use `videoconvert` to convert.

2. **USB Connection**: For stable streaming, connect the device to a USB host and open a camera application before starting GStreamer.

3. **Video Device**: UVC gadget typically appears as `/dev/video0` with name containing "gadget" or "dwc3".

4. **Kernel Limitation**: Each frame configuration supports only one frame interval. Create separate configurations for different frame rates (e.g., `1080p_30_yuy2` and `1080p_60_yuy2`).

## Troubleshooting

### "Device or resource busy"
- Stop any running GStreamer processes: `pkill gst-launch`
- Wait a few seconds before reconfiguring
- Use `uvc_config.sh restart` to fully reset

### "not-negotiated" error
- Ensure USB host is connected and requesting video
- Check format matches configured resolution and frame rate
- Verify UVC gadget is enabled: `uvc_config.sh status`

### Check UVC gadget status
```bash
uvc_config.sh status
cat /sys/kernel/config/usb_gadget/rockchip/UDC
ls /dev/video*
```
