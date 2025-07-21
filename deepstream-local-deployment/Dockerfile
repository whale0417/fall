# Step 1: Use the DeepStream image as the base
FROM nvcr.io/nvidia/deepstream:7.0-triton-multiarch

# Step 2: Install missing libraries and plugins
RUN apt-get update && apt-get install -y \
    libgstreamer1.0-dev \
    libgstreamer-plugins-base1.0-dev \
    libgstreamer-plugins-bad1.0-dev \
    gstreamer1.0-plugins-base \
    gstreamer1.0-plugins-good \
    gstreamer1.0-plugins-bad \
    gstreamer1.0-plugins-ugly \
    gstreamer1.0-libav \
    gstreamer1.0-tools \
    gstreamer1.0-x \
    gstreamer1.0-alsa \
    gstreamer1.0-gl \
    gstreamer1.0-gtk3 \
    gstreamer1.0-qt5 \
    gstreamer1.0-pulseaudio \
    libavcodec58 \
    libmpg123-0 \
    libmpeg2-4 \
    libmpeg2encpp-2.1-0 \
    libopenmpt0 \
    && rm -rf /var/lib/apt/lists/*

# Step 3: Copy custom configuration files (optional)
# If you have custom configuration files, copy them into the image
# COPY ./path/to/your/custom-config.txt /opt/nvidia/deepstream/deepstream/samples/configs/deepstream-app/

# Step 4: Set the default command to run the DeepStream app
CMD ["deepstream-app", "-c", "test_config.txt"]
