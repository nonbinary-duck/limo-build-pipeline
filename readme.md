## What is this and why?

This is a configuration which facilitates building some ROS2 Humble packages maintained by [LCAS](https://github.com/LCAS).

Those packages are typically provided in a convenient docker image which works well. The docker image is portable, easy to develop with and integrates support for the Zenoh ros2 DDS bridge so there truly is little need for `make`-your-own stuff. However, I enjoy wasting my time and so I have `make`ed-my-own.

## Build Instructions

This pipeline requires a recent version of Ubuntu. The instructions have been tested on an Ubuntu 22.04 (LTS) server live disk install image.

### System

First, the system must be setup.

Add the ROS2 PPA to Ubuntu if not already present.

You can do this following the [ROS Humble docs](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debians.html). Or just:

```bash
# Add universe
sudo apt install software-properties-common
sudo add-apt-repository universe

# Update our aptitude and install curl
# Ubuntu server 22.04 has curl and wget, but we do this for completeness
sudo apt update && sudo apt install curl

# curl the ros key
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg

# Add the ROS2 PPA (manually)
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

# Get that PPA for an update
sudo apt update
```

Then, some other packages are required to build our ROS packages. Most of them we can fetch from aptitude:
```bash
# Please paste me as a single line!
sudo apt install ros-humble-ros-base ros-dev-tools ros-humble-camera-ros ros-humble-libcamera nlohmann-json3-dev ros-humble-joint-state-* libusb-dev libusb-1.0-0-dev build-essential cmake pkg-config gcc python3 swig python3-pip git libgoogle-glog-dev ros-humble-image-geometry ros-humble-image-publisher
```

Now clone this repo for the submodules it contains:
```bash
# --recurse-submodules clones the submodules this git repo has recursively

# We can use ssh if you've got git setup with ssh keys:
git clone git@github.com:nonbinary-duck/limo-build-pipeline.git --recurse-submodules

# Or use https if not:
git clone https://github.com/nonbinary-duck/limo-build-pipeline.git --recurse-submodules


# How to setup git using ssh keys:

# Make a keypair (and store to the appropriate directory)
# Note that ~/.ssh should have 700 permission bits (no read access for group or other users!!)
ssh-keygen -t rsa -f ~/.ssh/id_github_rsa

# Make an ssh config file (warning, this will overwrite your existing config if you have one!!)
echo "Host github.com 
    IdentityFile ~/.ssh/id_github_rsa
" > ~/.ssh/config

# And then you'll need to paste your public key into github settings under "SSH keys"
# This will print your public key to console so you can copy it into the clipboard
cat ~/.ssh/id_github_rsa.pub
```

There are some things we can't get from aptitude, so we build them too! We've already installed the requirements for the packages under `./system-items` so we just need to build them.

Note that in Ubuntu 22.04 there is a `libuvc-dev` package, however the astra camera package fails to build under that version so we install it from source instead of using apt.

In no particular order, as none of these packages depend on another, use standard CMake build steps and finish with a system-level install for convenience:

```bash
# Start in the root directory of this cloned repo
cd ./system-items/libuvc && mkdir build && cd build && cmake ..

# Make libuvc with up to 20 (this number is meaningless) parallel jobs
make -j20 && sudo make install

# Make magic_enum, it fails using large job numbers sometimes
cd ../../magic_enum && mkdir build && cd build && cmake .. && make -j4 && sudo make install

# Make YDLidar-SDK
cd ../../YDLidar-SDK && mkdir build && cd build && cmake .. && make -j20 && sudo make install
```

### ROS Packages

Now we have all of the system components, we can finally build our ros packages:

```bash
# Start in the root directory of this cloned repo
cd ./ros-items/

# Make sure we've sourced ROS Humble
source /opt/ros/humble/setup.bash

# Build our packages
# If this command fails due to "Killed signal terminated program cc1plus", try running the command again
colcon build

# Source what we've built
# We won't be able to use it yet since we need gazebo, see next section!
# But you can inspect the packages
source install/setup.bash
```


## Use

To use these packages gazebo is required. An easy way to do this is just:
```bash
# It should work with just the gazebo packages turtlebot uses, or maybe just the base gazebo package
# But failing that you can just install all of it too with `sudo apt install ros-humble-gazebo*`
sudo apt install ros-humble-turtlebot3-gazebo
```

Additionally, some environment variables are required. Adding these lines to the end of your `~/.bashrc` file works:
```bash
export GAZEBO_MODEL_PATH="/usr/share/gazebo-11/models:"
export GAZEBO_PLUGIN_PATH="/usr/lib/x86_64-linux-gnu/gazebo-11/plugins:"
export GAZEBO_RESOURCE_PATH="/usr/share/gazebo-11:"
export GAZEBO_MODEL_DATABASE_URI="http://models.gazebosim.org"

# If you're using turtlebot3
export TURTLEBOT3_MODEL=waffle
```

And for automatically sourcing ros, add this to the `.bashrc` too:

```bash
source /opt/ros/humble/setup.bash
# Replace foo/bar with the path relative to your home directory where you have cloned this repo!
source ~/foo/bar/limo-build-pipeline/ros-items/install/setup.bash
```

To get that final foo/bar path you could also execute (and copy into your clipboard) this command, running in the same directory as this readme:

```bash
# Paste the result of this in your .bashrc!
echo "source ~/$(realpath --relative-base="$HOME" .)/ros-items/install/setup.bash"
```

Use of realpath taken from [this stack overflow answer](https://stackoverflow.com/a/62684928).


