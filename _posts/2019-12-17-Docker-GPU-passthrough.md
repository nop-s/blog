# Docker and NVIDIA GPU Passthrough

The release of Docker 19.03+ has builtin support for GPU passthrough with the `--gpus` directive. This change deprecates some packaging that NVIDIA had to enable passthrough for their GPUs and makes the entire process easier to use. However there are still a couple of gotchas. The "general" fix to enable access to the GPU for all capabilities is adding the following environment variables to the docker run/launch:
1. NVIDIA_DRIVER_CAPABILITIES=all
2. NVIDIA_VISIBLE_DEVICES=all
3. gpus=all

Or in easy cut and paste format:
```
        -e NVIDIA_DRIVER_CAPABILITIES=all \
        -e NVIDIA_VISIBLE_DEVICES=all \
	--gpus=all \
```

To check if the GPU is being used in the container, you can use `nvidia-smi dmon`, you should see activity scroll by on use.

This will also make hardware transcoding in Plex work.

Reference URLs:
1. <https://github.com/NVIDIA/nvidia-container-runtime>
2. <https://github.com/NVIDIA/nvidia-docker>
3. <https://developer.nvidia.com/video-encode-decode-gpu-support-matrix>
