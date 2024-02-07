# Raspberry Pi Instructions

## Building on Raspberry Pi OS (64-bit) Debian version: 12 (bookworm)

* For whatever reason `PyQt5`, `PyQt6`, `pyqtgraph` runs into a lot of issues.
  * `pip install PyQt5` (at the time of writing) installs `5.15.10`. The install hangs infinitely on `Preparing wheel metadata ... -`. When you run `pip install PyQt5 --verbose`, you see the install appears to be hanging infintely on a non-responsive license agreement. According to [this](https://stackoverflow.com/questions/73714829/pip-install-pyqt5-it-cannot-go-on/74071222#74071222) post, this seems to be an issue with `pip 20.0.2` (the default pip version in an `ubuntu:20.04` `venv` installed with `apt install python3-venv`). You can accept the license with `pip install pyqt5 --config-settings --confirm-license= --verbose`, but the `--config-settings` argument requires a newer version of `pip`. Running `pip install -U pip` seems to fix the issue. Alternatively, as stated in [this](https://github.com/pypa/pip/issues/12058) post, this can be fixed by downgrading to an older version of `pip install pyqt5==5.15.2`
  * `pip install PyQt6` (at the time of writing) installs `6.6.1` and crashes with `error: metadata-generation-failed` due to a `sipbuild.exceptions.UserException`.
  * `pip install pyqtgraph` (at the time of writing) installs version `0.13.3`, which has `AttributeError: module 'pyqtgraph.Qt.QtGui' has no attribute 'QApplication'`. As per [this](https://github.com/AdaptiveParticles/pyapr/issues/68) post, this issue is fixed by downgrading with `pip install pyqtgraph==0.12.4`.
* Since Raspberry Pi OS is currently on Debian 12, there are newer packages in `apt` than would be in Ubuntu 20.04 (e.g., `python 3.11`). Thus, we will install the needed python libraries through `apt`.
```bash
sudo apt install libboost-all-dev
sudo apt install python-is-python3 python3-pip python3-venv
sudo apt install python3-matplotlib python3-numpy python3-pyqtgraph python3-pyqt5 python3-pyqt6 python3-serial python3-scipy
```
* We modify `bindings/python/setup.py.in` to not have `make pip-install` try to install these packages with `pip`. We note `examples/plotfreq.py` seems to relies on `scipy` and `PyQt6` which weren't included in the original `install_requires`.
* We run into a build issue at `make` with the following error:
```bash
Building CXX object bindings/python/CMakeFiles/py_teensyimu.dir/py_teensyimu.cpp.o
error: invalid use of incomplete type ‘PyFrameObject’ {aka ‘struct _frame’}
```
* As per [this](https://github.com/pybind/pybind11/discussions/4333) post, this issue is fixed in `pybind11 2.10.4`. We modify `cmake/pybind11.cmake.in` to use a `GIT_TAG` of `v2.10.4`.
* Raspberry Pi OS is configured to not let you install packages directly with `pip`. Thus `make pip-install` says it successfully install `teensyimu`, but when you `pip list` afterwards it isn't present. To fix this we do:
  * Instead we install `teensyimu` to a `venv` (here with the name `venv`) with the following:
  ```bash
  python -m venv venv
  ```
  * Since we will be using system packages in conjunction with `venv`, we need to edit `venv/pyenv.cfg` to set:
  ```bash
  include-system-site-pacages = true
  ```
  * We can then activate our `venv` with:
  ```bash
  source venv/bin/activate
  ```
  * And build with `make pip-install`
  * As per [this](https://deycode.com/posts/how-to-solve-error-externally-managed-environment-when-using-pip3/) post, you can alternatively (and recklessly) override this with the following config at `~/.config/pip/pip.conf`:
  ```
  [global]
  break-system-packages = true
  ```
* To work correctly with `venv` the shebang in `examples/plotimu.py` and `examples/plotfreq.py` was changed to `/usr/bin/env python`.
* Using [esp32imu's plotimu.py](https://github.com/plusk01/esp32imu/blob/main/examples/plotimu.py) as a guide, updated `examples/plotimu.py` to use `PyQt6`.
* As is fixed in [esp32imu's plotfreq.py](https://github.com/plusk01/esp32imu/blob/main/examples/plotfreq.py), fixed a name collision for `self.window` by changing signal window to `self.sigwin` and leaving the Qt window as `self.window`.

### Full instructions
```bash
# install with apt
sudo apt install libboost-all-dev
sudo apt install python-is-python3 python3-pip python3-venv
sudo apt install python3-matplotlib python3-numpy python3-pyqtgraph python3-pyqt5 python3-pyqt6 python3-serial python3-scipy

# clone forked repo
git clone git@github.com:fishberg/teensyimu.git
cd teensyimu

# setup venv
python -m venv venv
vi venv/pyenv.cfg
# edit -> `include-system-site-pacages = true`
source venv/bin/activate

# build teensyimu
mkdir build
cd build
cmake ..
make -j
make pip-install

# test package
python -m teensyimu.plotimu
python -m teensyimu.plotfreq

# test scripts
cd ../examples
python plotimu.py
python plotfreq.py
```

## Install Teensy Arduino Enviornment
* Arduino IDE 2.X is based on Electron and does not (currently) run on ARM processors.
* Although we can install Arduino 1.8 via `apt`, we need to modify the installation so we can flash Teensy. Thus Arduino cannot be managed by our package manager. (Installing extra boards is better in 2.X, but that isn't helpful here.)
* You can download Arduino 1.8 as a legacy install [here](https://downloads.arduino.cc/arduino-1.8.19-linuxaarch64.tar.xz).
* Follow the instructions [here](https://www.pjrc.com/teensy/td_download.html) to install Teensyduino for Arduino 1.8.
    * `udev` rules can be downloaded [here](https://www.pjrc.com/teensy/00-teensy.rules). The rules can be installed and loaded with:
    ```bash
    sudo cp 00-teensy.rules /etc/udev/rules.d/
    sudo udevadm control --reload-rules && sudo udevadm trigger
    ```
    * Use [this](https://www.pjrc.com/teensy/td_158/TeensyduinoInstall.linuxaarch64) Teensyduino installer. Note we are using the `ARM 64 bit / AARCH64 / Jetson TX2` version and NOT the `ARM 32 bit / Raspberry Pi` version -- Raspberry Pi OS now runs on 64 bit and the page hasn't been updated. When you run the install, you will need to use the GUI to navigate to the Arduino 1.8 folder so it can modify the files. You can run the installer with:
    ```bash
    chmod 755 TeensyduinoInstall.linux64
    ./TeensyduinoInstall.linux64
    ```
* Configuring Arduino 1.8:
  * Install `ICM_20948.h` [here](http://librarymanager/All#SparkFun_ICM_20948_IMU) with `Tools > Manage Libraries`
  * Set the board to `Board > Teensyduino > Teensy 4.0` (installed by `TeensyduinoInstall.linux64`)
  * Set the Port to `Port > Teensy Ports > /dev/ttyACMx` where `x` is the correct port (installed by `TeensyduinoInstall.linux64`)
