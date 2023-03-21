# *** fork for AIRT support ***

## install prerequisites

Ensure you have the latest docker installed, and also qemu:

```apt-get install -y qemu-user-static```

Build gnuradio with soapy frequency tuning command patches, on x86 host,
if you do not already have the ```anarkiwi-airt-gnuradio``` channel.

```
$ git clone https://github.com/anarkiwi/gnuradio-feedstock -b airt3
$ cd gnuradio-feedstock
$ ./build-locally.py
```

choose option 5 (```linux_aarch64_numpy1.20python3.8.____cpython```).

tar up ```build_artifacts``` and copy into $HOME on AIRT, as ```anarkiwi-airt-gnuradio```.
*NOTE:* ensure you do not have $DISPLAY set during the build to avoid a hang.

Ensure you have airstack 0.5.5rc2 or later .deb installed (need tuning fixes).

Configure xdma (otherwise tuning will be very slow).

```
$ sudo rmmod xdma
$ sudo modprobe xdma rx_buffer_size_pages=1024
```

Create anarkiwi-airt.yml

```
name: anarkiwi-airt
channels:
  - /home/anarkiwi/anarkiwi-airt-gnuradio
  - file://opt/deepwave/conda-channels/airstack-conda
  - conda-forge
  - nvidia
  - defaults

dependencies:
  - python=3.8
  - cmake
  - scipy
  - numpy
  - matplotlib
  - pip
  - pybind11
  - soapysdr-module-airt=0.5.5rc2
  - gnuradio=3.9.8
```

create conda environment

```
$ conda env create -f anarkiwi-airt.yml
```

activate conda

```
$ conda activate anarkiwi-airt
```

install gr-iqtlabs

```
$ git clone https://github.com/iqtlabs/gr-iqtlabs -b 1.0.15
$ cd gr-iqtlabs
$ mkdir build
$ cd build
$ cmake ..
$ make
$ sudo make install
$ ln -s /usr/local/lib/libgnuradio-iqtlabs.so* ~/.conda/envs/anarkiwi-airt/lib
$ ln -s /usr/local/lib/python3.8/site-packages/gnuradio/iqtlabs ~/.conda/envs/anarkiwi-airt/lib/python3.8/site-packages/gnuradio
```

install gr-wavelearner

```
$ git clone https://github.com/deepwavedigital/gr-wavelearner
$ cd gr-wavelearner
$ mkdir build
$ cd build
$ cmake ..
$ make
$ sudo make install
$ ln -s /usr/local/lib/python3.8/site-packages/wavelearner/ ~/.conda/envs/anarkiwi-airt/lib/python3.8
$ ln -s /usr/local/lib/libgnuradio-wavelearner.so* ~/.conda/envs/anarkiwi-airt/lib/
```

install gamutrf

```
$ git clone https://github.com/iqtlabs/gamutrf
$ cd gamutrf
$ pip3 install .
```

run gamutrf (may need to change sample rate depending on SDR).

```
$ gamutrf-scan --sdr=SoapyAIRT --freq-start=300e6 --freq-end=6e9 --tune-step-fft 1024 --samp-rate=100e6 --nfft 256 --tuneoverlap 1
$ gamutrf-sigfinder --promport=9009 --fftgraph fft.png --port 9005 --nfftplots 0 --db_rolling_factor 0 --buff_path $HOME
```

gamutrf-scan will repeatedly print

```
gr::log :ERROR: source0 - soapy: ignoring unknown command key 'tag'
gr::log :DEBUG: retune_fft0 - rx_freq: {}
gr::log :DEBUG: retune_fft0 - new rx_freq tag: {}, last {}
gr::log :DEBUG: retune_fft0 - retuning to {}
```

while in operation - this is normal, the broken logging is because gamutrf/gr-iqtlabs are written for gnuradio 3.10, not for gnuradio 3.9 style logging.