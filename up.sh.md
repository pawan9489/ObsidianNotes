> chmod +x ~/up.sh
> ~/up.sh


```sh
#!/bin/bash

echo "Starting DNF and Firmware Updates..."

# Step 1: Update DNF packages with a fresh repository refresh
echo "Step 1/5: Running DNF upgrade with a forced refresh..."
sudo dnf upgrade --refresh -y

# Step 2: Remove unneeded packages
echo "Step 2/5: Cleaning up old packages..."
sudo dnf autoremove -y

# Step 3: Refresh firmware data
echo "Step 3/5: Refreshing firmware data..."
sudo fwupdmgr refresh --force

# Step 4: Get a list of available firmware updates
echo "Step 4/5: Getting a list of available firmware updates..."
sudo fwupdmgr get-updates

# Step 5: Install firmware updates
echo "Step 5/5: Installing firmware updates..."
sudo fwupdmgr update

echo "All updates attempted!"

# Check if a reboot is needed for firmware updates
if sudo fwupdmgr get-devices --show-reboot-required | grep "needs a reboot"; then
    echo "⚠️ A system reboot is required for firmware updates to take effect."
    echo "Please reboot your system now."
else
    echo "✅ No reboot required for firmware updates."
fi
```

Upgrade the fedora versions
> sudo dnf install dnf-plugin-system-upgrade


```
# Download all new packages
sudo dnf system-upgrade download --releasever=43

# Install new package and 
sudo dnf system-upgrade reboot
```