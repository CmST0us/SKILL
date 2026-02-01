# UVC Gadget Skill

This skill provides knowledge about configuring and testing USB Video Class (UVC) gadget on Rockchip platforms.

## UVC Gadget App
`https://gitlab.freedesktop.org/camera/uvc-gadget`

### Configure Script
see:
`https://gitlab.freedesktop.org/camera/uvc-gadget/-/blob/master/scripts/uvc-gadget.sh?ref_type=heads`


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
