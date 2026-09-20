# Proxmox GPU Passthrough & Local LLM Deployment Guide

## Hardware & Environment Specifications
*   **Host Hypervisor:** Proxmox VE 9.2.2 (Enterprise Kernel `7.0.x`)
*   **Host CPU:** Intel Core i5-6600 (with Intel HD Graphics P530 integrated graphics)
*   **Dedicated GPU:** NVIDIA Quadro P620 (Pascal Architecture, 2GB VRAM)
*   **Guest OS:** Ubuntu 26.04 LTS (Headless Server)
*   **Network:** VM IP `<YOUR_VM_IP>`
*   **Target Application:** Ollama (Port `11434`) and Open WebUI (Port `3000`)

---

## Phase 1: Proxmox Host Configuration (VFIO Isolation)
Instruct Proxmox to release control of the hardware. **Run these commands on the Proxmox Host terminal.**

1.  **Enable IOMMU in the Bootloader**
    Open the GRUB configuration file:
    ```bash
    nano /etc/default/grub
    ```
    Modify the command line to include the IOMMU flags:
    ```text
    GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
    ```
    Update the bootloader:
    ```bash
    update-grub
    ```

2.  **Load Virtualization Modules**
    Inject the required VFIO kernel modules:
    ```bash
    echo -e "vfio\nvfio_iommu_type1\nvfio_pci" >> /etc/modules
    ```

3.  **Blacklist Open-Source Drivers**
    Prevent Proxmox from loading default display drivers:
    ```bash
    echo -e "blacklist nouveau\nblacklist nvidia\nblacklist nvidiafb" > /etc/modprobe.d/blacklist.conf
    ```

4.  **Bind VFIO to the GPU**
    Lock the virtualization driver to the Quadro P620 hardware IDs (`10de:1cb6` for VGA, `10de:0fb9` for Audio):
    ```bash
    echo "options vfio-pci ids=10de:1cb6,10de:0fb9 disable_vga=1" > /etc/modprobe.d/vfio.conf
    ```

5.  **Apply and Reboot**
    Rebuild the boot image and restart the host:
    ```bash
    update-initramfs -u -k all
    reboot
    ```
    Verify the GPU is using the `vfio-pci` driver:
    ```bash
    lspci -nnk | grep -i vga -A 3
    ```

---

## Phase 2: Virtual Machine Setup & Driver Installation
Provision the VM environment. **Run the following commands inside the Ubuntu VM terminal.**

1.  **VM Hardware Provisioning (via Proxmox UI)**
    *   **Machine Type:** `q35`
    *   **BIOS:** `OVMF (UEFI)`
    *   **CPU Type:** `host`
    *   **PCI Device:** Add -> PCI Device -> Select the Quadro P620. Check **All Functions**, **ROM-Bar**, and **PCI-Express**.

2.  **Fix Network DNS Resolution**
    Force the VM to use Google's public DNS servers to resolve package download failures:
    ```bash
    sudo sed -i 's/.*DNS=.*/DNS=8.8.8.8 8.8.4.4/' /etc/systemd/resolved.conf
    sudo systemctl restart systemd-resolved
    ping -c 4 google.com
    ```

3.  **Install Legacy NVIDIA Drivers**
    *Challenge:* NVIDIA dropped Pascal architecture support in the `590+` driver branches. The legacy `580` branch must be used.
    ```bash
    sudo apt purge "^libnvidia-.*" "^nvidia-.*" -y
    sudo apt autoremove -y
    sudo apt update
    sudo apt install nvidia-driver-580-server nvidia-utils-580-server -y
    ```

4.  **Disable Secure Boot (via Proxmox UI)**
    *Challenge:* Ubuntu's default Secure Boot blocks unsigned proprietary NVIDIA kernel modules.
    *   Run `sudo reboot` in the VM.
    *   Rapidly press **ESC** in the Proxmox VM console to enter the virtual BIOS.
    *   Navigate to **Device Manager** -> **Secure Boot Configuration**.
    *   Uncheck **Attempt Secure Boot**. Press **F10** to save, **Y** to confirm, and reboot.
    *   Verify GPU acceleration in Ubuntu:
    ```bash
    nvidia-smi
    ```

---

## Phase 3: Local LLM Deployment & Network Exposure
Install the AI engine and expose it to the local network. **Run these commands in the Ubuntu VM.**

1.  **Install Ollama**
    ```bash
    curl -fsSL [https://ollama.com/install.sh](https://ollama.com/install.sh) | sh
    ```

2.  **Expose to Local Network**
    By default, Ollama only listens to `localhost`. Inject a `0.0.0.0` host binding rule:
    ```bash
    sudo mkdir -p /etc/systemd/system/ollama.service.d
    echo -e "[Service]\nEnvironment=\"OLLAMA_HOST=0.0.0.0\"" | sudo tee /etc/systemd/system/ollama.service.d/override.conf
    ```

3.  **Apply and Restart**
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl restart ollama
    ```
    *Verification:* On another computer, navigate to `http://<YOUR_VM_IP>:11434`. It should display "Ollama is running".

---

## Phase 4: Open WebUI (Graphical Interface) Deployment
Deploy the ChatGPT-like interface via Docker as a `systemd` background service. **Run these commands in the Ubuntu VM.**

1.  **Install Docker Engine**
    ```bash
    curl -fsSL [https://get.docker.com](https://get.docker.com) -o get-docker.sh
    sudo sh get-docker.sh
    ```

2.  **Create the Container Environment**
    Prepare the Docker container, linking it to the host network gateway so it can communicate with Ollama:
    ```bash
    sudo docker create -p 3000:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:main
    ```

3.  **Create the Systemd Service**
    Write a custom service file forcing the UI to wait for Ollama to boot first (`After=docker.service ollama.service`). Run this block as one command:
    ```bash
    cat << 'EOF' | sudo tee /etc/systemd/system/open-webui.service
    [Unit]
    Description=Open WebUI Docker Container
    Requires=docker.service
    After=docker.service ollama.service

    [Service]
    Restart=always
    ExecStart=/usr/bin/docker start -a open-webui
    ExecStop=/usr/bin/docker stop -t 2 open-webui

    [Install]
    WantedBy=multi-user.target
    EOF
    ```

4.  **Enable and Start the UI**
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable open-webui
    sudo systemctl start open-webui
    ```
    *Access:* The UI is available network-wide at `http://<YOUR_VM_IP>:3000`.

---

## Phase 5: Hardware Optimization & Troubleshooting

**VRAM Constraint:**
The Quadro P620 is limited to **2GB of VRAM**. Attempting to run standard 8B parameter models will cause severe memory spillover to system RAM, resulting in extremely slow generation or crashes.

**Recommended Models:**
*   `qwen2.5:1.5b` (Best overall intelligence/coding for <2GB)
*   `llama3.2:1b` (Fastest for text generation/summarization)

**Known Issues & Resolutions:**
*   **Error: "does not support tools" (TinyLlama)**
    *   *Cause:* Older or highly simplified micro-models lack the architecture for Open WebUI's advanced function calling (like Web Search).
    *   *Resolution:* Turn off Web Search/Tools in the UI chat bar before sending a message, or switch to a modern model like `qwen2.5:1.5b`.
*   **Error: "Oops! There was an error in the previous response."**
    *   *Cause:* The UI frontend de-synced from the backend, or Ollama ran out of VRAM while trying to swap models.
    *   *Resolution:* Flush the GPU memory by restarting the engine via SSH (`sudo systemctl restart ollama`), then open a **New Chat** window in Open WebUI to clear the corrupted frontend history.
