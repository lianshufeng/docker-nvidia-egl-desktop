# docker-nvidia-egl-desktop

MATE Desktop container designed for Kubernetes with direct access to the GPU with EGL using VirtualGL for GPUs with WebRTC and HTML5, providing an open source remote cloud graphics or game streaming platform. Does not require `/tmp/.X11-unix` host sockets or host configuration.

Use [docker-nvidia-glx-desktop](https://github.com/lianshufeng/docker-nvidia-glx-desktop) for a MATE Desktop container with better performance, also including Vulkan support for NVIDIA GPUs by spawning its own X Server without using `/tmp/.X11-unix` host sockets.

### Usage

This container is composed fully of vendor-neutral applications and protocols except the NVIDIA base container itself, meaning that there is nothing stopping you from using this container with GPUs of other vendors including AMD and Intel (as a hint use the respective vendor's container runtime or Kubernetes device plugin and make sure that it provisions `/dev/dri/cardX` devices, then set `WEBRTC_ENCODER` to the value `x264enc` if using the selkies-gstreamer WebRTC interface). However, this is not officially supported as the maintainers only use NVIDIA hardware. This container also supports running without any GPUs with software fallback (set `WEBRTC_ENCODER` to the value `x264enc` if using the selkies-gstreamer WebRTC interface).

Wine, Winetricks, and PlayOnLinux are bundled by default. Comment out the section where it is installed within `Dockerfile` if the user wants to remove them from the container.

There are two web interfaces that can be chosen in this container, the first being the default [selkies-gstreamer](https://github.com/selkies-project/selkies-gstreamer) WebRTC HTML5 interface, and the second being the fallback [noVNC](https://github.com/novnc/noVNC) WebSocket HTML5 interface. The noVNC interface can be enabled by setting `NOVNC_ENABLE` to `true`. While the noVNC interface does not support audio forwarding, it can be useful for troubleshooting the selkies-gstreamer WebRTC interface or using this container with low bandwidth environments. When using the noVNC interface, all environment variables related to the selkies-gstreamer WebRTC interface are ignored, with the exception of `BASIC_AUTH_PASSWORD`. As with the selkies-gstreamer WebRTC interface, the noVNC interface password will be set to `BASIC_AUTH_PASSWORD`, and use `PASSWD` by default if not set. The noVNC interface also additionally accepts the `NOVNC_VIEWPASS` environment variable, where a view only password with only the ability to observe the desktop passively can also be provisioned.

The container requires host NVIDIA GPU driver versions of at least **450.80.02** (unless when using software acceleration fallback without allocated GPUs), with the corresponding container toolkit runtime to be also configured on the host for allocating GPUs. All Maxwell or later generation GPUs in the consumer, professional, or datacenter lineups will not have significant issues running this container, although the selkies-gstreamer high performance NVENC backend may not be available (see the next paragraph). Kepler GPUs are untested and likely does not support the NVENC backend, but can be mostly functional using the software acceleration fallback.

The high performance NVENC backend for the selkies-gstreamer WebRTC interface is only supported in GPUs listed as supporting `H.264 (AVCHD)` under the `NVENC - Encoding` section of NVIDIA's [Video Encode and Decode GPU Support Matrix](https://developer.nvidia.com/video-encode-and-decode-gpu-support-matrix-new). If you are using software fallback without allocated GPUs or your GPU is not listed as supporting `H.264 (AVCHD)`, add the environment variable `WEBRTC_ENCODER` with the value `x264enc` in your container configuration for falling back to software acceleration, which also has a very good performance depending on your CPU.

The username is `user` in both the container user account and the web authentication prompt. The environment variable `PASSWD` is the password of the container user account, and `BASIC_AUTH_PASSWORD` is the password for the HTML5 interface authentication prompt. If `ENABLE_BASIC_AUTH` is set to `true` for selkies-gstreamer (not required for noVNC) but `BASIC_AUTH_PASSWORD` is unspecified, the HTML5 interface password will default to `PASSWD`.
> NOTES: Only one web browser can be connected at a time with the selkies-gstreamer WebRTC interface. If the signaling connection works, but the WebRTC connection fails, read the [Using a TURN Server](#using-a-turn-server) section.

#### Running with Docker

1. Run the container with Docker (or other similar container CLIs like Podman):

```
docker run --gpus 1 -it -e TZ=UTC -e SIZEW=1920 -e SIZEH=1080 -e REFRESH=60 -e DPI=96 -e CDEPTH=24 -e PASSWD=mypasswd -e WEBRTC_ENCODER=nvh264enc -e BASIC_AUTH_PASSWORD=mypasswd -p 8080:8080 lianshufeng/desktop:latest
```
> NOTES: The container tags available are `latest` and `20.04` for Ubuntu 20.04 and `18.04` for Ubuntu 18.04. Replace all instances of `mypasswd` with your desired password. `BASIC_AUTH_PASSWORD` will default to `PASSWD` if unspecified. The container must not be run in privileged mode. Use the option `--tmpfs /dev/shm:rw` for a slight performance improvement.

Change `WEBRTC_ENCODER` to `x264enc` when using the selkies-gstreamer interface if you are using software fallback without allocated GPUs or your GPU doesn't support `H.264 (AVCHD)` under the `NVENC - Encoding` section in NVIDIA's [Video Encode and Decode GPU Support Matrix](https://developer.nvidia.com/video-encode-and-decode-gpu-support-matrix-new).

2. Connect to the web server with a browser on port 8080. You may also separately configure a reverse proxy to this port for external connectivity.
> NOTES: Additional configurations and environment variables for the selkies-gstreamer WebRTC HTML5 interface are listed in lines that start with `parser.add_argument` within the [selkies-gstreamer main script](https://github.com/selkies-project/selkies-gstreamer/blob/master/src/selkies_gstreamer/__main__.py).

#### Running with Kubernetes

1. Create the Kubernetes Secret with your authentication password:

```bash
kubectl create secret generic my-pass --from-literal=my-pass=YOUR_PASSWORD
```
> NOTES: Replace `YOUR_PASSWORD` with your desired password, and change the name `my-pass` to your preferred name of the Kubernetes secret with the `egl.yml` file changed accordingly as well. It is possible to skip the first step and directly provide the password with `value:` in `egl.yml`, but this exposes the password in plain text.

2. Create the pod after editing the `egl.yml` file to your needs:

```bash
kubectl create -f egl.yml
```
> NOTES: The container tags available are `latest` and `20.04` for Ubuntu 20.04 and `18.04` for Ubuntu 18.04. `BASIC_AUTH_PASSWORD` will default to `PASSWD` if unspecified.

Change `WEBRTC_ENCODER` to `x264enc` when using the selkies-gstreamer interface if you are using software fallback without allocated GPUs or your GPU doesn't support `H.264 (AVCHD)` under the `NVENC - Encoding` section in NVIDIA's [Video Encode and Decode GPU Support Matrix](https://developer.nvidia.com/video-encode-and-decode-gpu-support-matrix-new).

3. Connect to the web server spawned at port 8080. You may configure the ingress endpoint or reverse proxy that your Kubernetes cluster provides to this port for external connectivity.
> NOTES: Additional configurations and environment variables for the selkies-gstreamer WebRTC HTML5 interface are listed in lines that start with `parser.add_argument` within the [selkies-gstreamer main script](https://github.com/selkies-project/selkies-gstreamer/blob/master/src/selkies_gstreamer/__main__.py).

#### Using a TURN Server

Note that this section is only required for the selkies-gstreamer WebRTC HTML5 interface. For an easy fix to when the signaling connection works, but the WebRTC connection fails, add the option `--network=host` to your Docker command, or uncomment `hostNetwork: true` in your `egl.yml` file when using Kubernetes (note that your cluster may have not allowed this, resulting in an error). This exposes your container to the host network, which disables network isolation. If this does not fix the connection issue (normally when the host is behind another firewall) or you do not want to use this fix for security or technical reasons, read the below text.

In most cases when either of your server or client has a permissive firewall, the default Google STUN server configuration will work without additional configuration. However, when connecting from networks that cannot be traversed with STUN, a TURN server is required. Provide the TURN server address, port, and shared secret in order to take advantage of the TURN relay capabilities and improve connection success.

An open source TURN server that can be used is [coTURN](https://github.com/coturn/coturn), and an [example container implementation](https://github.com/selkies-project/selkies-gstreamer/tree/master/addons/coturn) `ghcr.io/selkies-project/selkies-gstreamer/coturn:latest` is available. For dynamic IP addresses, [dynamic-coturn](https://github.com/mreichardt95/dynamic-coturn) is a container implementation which restarts the TURN server whenever the public IP address gets changed. [Pion TURN](https://github.com/pion/turn) is another TURN server implementation compatible with all major operating systems, and [restund](https://openwrt.org/packages/pkgdata/restund) is a TURN server implementation for OpenWRT.

The [Numb STUN/TURN Server](https://numb.viagenie.ca) is a free TURN server instance that may be used for personal purposes upon registration, but may not be optimal for production usage.

With Docker, use the `-e` option to add the `TURN_HOST`, `TURN_PORT` environment variables. You also require to provide either just `TURN_SHARED_SECRET` for time-limited shared secret TURN authentication, or both `TURN_USERNAME` and `TURN_PASSWORD` for legacy long term TURN authentication, depending on your TURN server configuration. Provide just one of these authentication methods, not both.

##### Configuring With Kubernetes

Your TURN server will use only one out of two ways to authenticate the client, so only provide one type of authentication method. The time-limited shared secret TURN authentication requires to only provide the Base64 encoded `TURN_SHARED_SECRET`. The legacy long term TURN authentication requires to provide both `TURN_USERNAME` and `TURN_PASSWORD` credentials.

###### Time-Limited Shared Secret Authentication

1. Create a secret containing the TURN shared secret:

```bash
kubectl create secret generic turn-shared-secret --from-literal=turn-shared-secret=MY_TURN_SHARED_SECRET
```
> NOTES: Replace `MY_TURN_SHARED_SECRET` with the shared secret of the TURN server, then changing the name `turn-shared-secret` to your preferred name of the Kubernetes secret, with the `egl.yml` file also being changed accordingly.

2. Uncomment the lines in the `egl.yml` file related to TURN server usage, updating the `TURN_HOST` and `TURN_PORT` environment variable as needed:

```yaml
- name: TURN_HOST
  value: "turn.example.com"
- name: TURN_PORT
  value: "3478"
- name: TURN_SHARED_SECRET
  valueFrom:
    secretKeyRef:
      name: turn-shared-secret
      key: turn-shared-secret
```
> NOTES: It is possible to skip the first step and directly provide the shared secret with `value:`, but this exposes the shared secret in plain text.

###### Legacy Long Term Authentication

1. Create a secret containing the TURN password:

```bash
kubectl create secret generic turn-password --from-literal=turn-password=MY_TURN_PASSWORD
```
> NOTES: Replace `MY_TURN_PASSWORD` with the password of the TURN server, then changing the name `turn-password` to your preferred name of the Kubernetes secret, with the `egl.yml` file also being changed accordingly.

2. Uncomment the lines in the `egl.yml` file related to TURN server usage, updating the `TURN_HOST`, `TURN_PORT`, and `TURN_USERNAME` environment variable as needed:

```yaml
- name: TURN_HOST
  value: "turn.example.com"
- name: TURN_PORT
  value: "3478"
- name: TURN_USERNAME
  value: "username"
- name: TURN_PASSWORD
  valueFrom:
    secretKeyRef:
      name: turn-password
      key: turn-password
```
> NOTES: It is possible to skip the first step and directly provide the TURN password with `value:`, but this exposes the TURN password in plain text.

---
**Please post issues relevant to the selkies-gstreamer WebRTC HTML5 interface to the [selkies-gstreamer](https://github.com/selkies-project/selkies-gstreamer) repository.**

This project involved a collaboration effort with [Dan Isla](https://github.com/danisla) of the [Selkies Project](https://github.com/selkies-project), incorporating the [selkies-gstreamer](https://github.com/selkies-project/selkies-gstreamer) WebRTC remote desktop streaming application.

This work was supported in part by NSF awards CNS-1730158, ACI-1540112, ACI-1541349, OAC-1826967, the University of California Office of the President, and the University of California San Diego’s California Institute for Telecommunications and Information Technology/Qualcomm Institute. Thanks to CENIC for the 100Gbps networks.
