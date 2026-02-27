# Stream DVR Prototype

This prototype demonstrates how to implement a streaming video player with sliding window DVR (Digital Video Recorder) capabilities using HTTP Live Streaming (HLS) and Video.js.

## How it works

1. **Ingest & Process (`camera-mock` service)**
   - To simulate your Dahua IP cameras, we run an `ffmpeg` container.
   - It generates a live mock stream (`testsrc`) at 30 fps.
   - It encodes the video as H.264 and packages it into HLS segments (`.ts` files).
   - Crucially, it uses a **Sliding Window Playlist** (`-hls_list_size 60`), which tells HLS to keep the last 60 segments (each 2 seconds = 120 seconds of history). This makes it possible to seek backward in the live stream!

2. **Server (`web` service)**
   - A lightweight NGINX server hosts the HLS segments (`html/hls/`) and the user interface (`html/index.html`).

3. **Web Interface (`index.html`)**
   - Built with `Video.js` (an industry-standard web player).
   - A custom UI overlay shows whether the user is at the "LIVE" edge or viewing "DVR / DELAYED" history.
   - Includes quick action buttons to instantly jump back 10 or 60 seconds, and a button to return to the Live edge.

## Running the Demo

1. Start the backend:
   ```bash
   docker compose up -d
   ```
2. Open your browser and navigate to:
   [http://localhost:8080](http://localhost:8080)

> Note: It may take 5–10 seconds for the first HLS segments to generate upon startup. If the player spins initially, just wait a few seconds, and it will load automatically.

## Integrating with your Dahua Cameras

To use this with your actual Dahua IP cameras, you just need to replace the `camera-mock` FFmpeg source input with the RTSP URL of your camera.

Modify `docker-compose.yml` to look something like this:

```yaml
  ffmpeg-camera1:
    image: jrottenberg/ffmpeg:4.4-alpine
    restart: unless-stopped
    command: >
      -rtsp_transport tcp -i "rtsp://admin:password@192.168.1.100:554/cam/realmonitor?channel=1&subtype=0"
      -c:v copy -c:a aac
      -f hls -hls_time 2 -hls_list_size 3600 -hls_flags delete_segments+append_list
      /output/camera1.m3u8
    volumes:
      - ./html/hls/camera1:/output
```

Notice `-c:v copy`: By copying the original video stream (instead of re-encoding it), it uses almost zero CPU! The `-hls_list_size 3600` parameter keeps the last 2 hours of video (3600 * 2 seconds = 7200s).

Then in your frontend, just point the Video.js source to `/hls/camera1/camera1.m3u8`!
