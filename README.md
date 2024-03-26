# Local LLaMA Server Setup Documentation

_TL;DR_: A guide to setting up a server for running local language models using `ollama`.

## About

This repository outlines the steps to run a server for running local language models. It uses Debian specifically, but most Linux distros should follow a very similar process. It aims to be a guide for Linux beginners like me who are setting up a server for the first time.

The process involves installing the NVIDIA drivers, setting the GPU power limit, and configuring the server to run `ollama` at boot. It also includes setting up auto-login and scheduling the `init.bash` script to run at boot. All these settings are based on my ideal setup for a language model server that runs most of the day but a lot can be customized to suit your needs. For example, you can use any OpenAI-compatible server like `llama.cpp` or LM Studio instead of `ollama`.

## System Requirements

This guide assumes that we're working with a system with one or more Nvidia GPUs and an Intel CPU. It should be identical for an AMD CPU (but I haven't verified this). 

The same cannot be said for AMD GPUs. Specifically for `ollama`, [there's work being done](https://github.com/ollama/ollama/issues/738#issuecomment-1974874171) to support ROCm but it still isn't stable in the way most people would hope it is for a reliable server. However, used Nvidia cards are a good deal nowadays (as of Mar 2024).

This guide was built around the following system:
- CPU: Intel Core i5-12600KF
- Memory: 32GB 6000 MHz DDR5 RAM
- Storage: 1TB M.2 NVMe SSD
- GPU: Nvidia RTX 3090 24GB

## Steps

1. #### Install Debian on the server
    - Download the [Debian ISO](https://www.debian.org/distrib/) from the official website.
    - Create a bootable USB using a tool like [Rufus](https://rufus.ie/en/) for Windows or [Balena Etcher](https://etcher.balena.io) for MacOS.
    - Boot into the USB and install Debian.

2. #### Update the system
    - Run the following command:
        ```
        sudo apt update
        ```

3. #### Install the NVIDIA drivers
    - Run the following commands:
        ```
        apt install linux-headers-amd64
        apt install nvidia-driver firmware-misc-nonfree
        ```
    - Reboot the server.
    - Run the following command to verify the installation:
        ```
        nvidia-smi
        ```

4. #### Install `ollama`
    - Download `ollama` from the official repository:
        ```
        curl -fsSL https://ollama.com/install.sh | sh
        ```
    - (Recommended) We want our LLM API endpoint to be reachable by the rest of the LAN. For `ollama`, this means setting `OLLAMA_HOST=0.0.0.0` in the `ollama.service`. 
      - Run the following command to edit the service:
        ```
        systemctl edit ollama.service
        ```
      - Find the `[Service]` section and add `Environment="OLLAMA_HOST=0.0.0.0"` under it. It should look like this:
        ```
        [Service]
        Environment="OLLAMA_HOST=0.0.0.0"
        ```
       - Save and exit.
       - Reload the environment.
            ```
            systemctl daemon-reload
            systemctl restart ollama
            ```
    
5. #### Create the `init.bash` script

    This script will be run at boot to set the GPU power limit and start the server using `ollama`/`llama.cpp`. We set the GPU power limit lower because it has been seen in testing and inference that there is only a 5-15% performance decrease for a 50% reduction in power consumption. This is especially important for servers that are running 24/7.

    - Run the following commands:
        ```
        touch init.bash
        nano init.bash
        ```
    - Add the following lines to the script:
        ```
        #!/bin/bash
        sudo nvidia-smi -pm 1
        sudo nvidia-smi -pl (power limit)
        ollama run (model)
        ollama serve
        ```
        > Replace `(power limit)` with the desired power limit in watts. For example, `sudo nvidia-smi -pl 250`.

        > Replace `(model)` with the name of the model you want to run. For example, `ollama run mistral:latest`.
    - Save and exit the script.
    - Make the script executable:
        ```
        chmod +x init.bash
        ```

6. #### Give `nvidia-persistenced` and `nvidia-smi` passwordless sudo permissions

    We want `init.bash` to run the `nvidia-smi` commands without having to enter a password. This is done by editing the `sudoers` file.

    - Run the following command:
        ```
        sudo visudo
        ```
    - Add the following lines to the file:
        ```
        (username) ALL=(ALL) NOPASSWD: /usr/bin/nvidia-persistenced
        (username) ALL=(ALL) NOPASSWD: /usr/bin/nvidia-smi
        ```
        > Replace `(username)` with your username.

        > **IMPORTANT**: Ensure that you add these lines AFTER `%sudo ALL=(ALL:ALL) ALL`. The order of the lines in the file matters - the last matching line will be used so if you add these lines before `%sudo ALL=(ALL:ALL) ALL`, they will be ignored.
    - Save and exit the file.

7. #### Configure auto-login

    When the server boots up, we want it to automatically log in to a user account and run the `init.bash` script. This is done by configuring the `lightdm` display manager.

    - Run the following command:
        ```
        sudo nano /etc/lightdm/lightdm.conf
        ```
    - Find the following commented line. It should be in the `[Seat:*]` section.
        ```
        # autologin-user=
        ```
    - Uncomment the line and add your username:
        ```
        autologin-user=(username)
        ```
        > Replace `(username)` with your username.
    - Save and exit the file.

8. #### Add `init.bash` to the crontab

    Adding the `init.bash` script to the crontab will schedule it to run at boot.

    - Run the following command:
        ```
        crontab -e
        ```
    - Add the following line to the file:
        ```
        @reboot /path/to/init.bash
        ```
        > Replace `/path/to/init.bash` with the path to the `init.bash` script.
    
    - (Optional) Add the following line to shutdown the server at 12am:
        ```
        0 0 * * * /sbin/shutdown -h now
        ```
    - Save and exit the file.

9. #### (Optional) Enable SSH

    Enabling SSH allows you to connect to the server remotely.

    On the server:
    - Run the following command:
        ```
        sudo apt install openssh-server
        ```
    - Start the SSH service:
        ```
        sudo systemctl start ssh
        ```
    - Enable the SSH service to start at boot:
        ```
        sudo systemctl enable ssh
        ```
    - Find the server's IP address:
        ```
        ip a
        ```

    On the client:
    - Connect to the server using SSH:
        ```
        ssh (username)@(ip address)
        ```
        > Replace `(username)` with your username and `(ip address)` with the server's IP address.

    If you expect to tunnel into your server often, I recommend following [this guide](https://www.raspberrypi.com/documentation/computers/remote-access.html#configure-ssh-without-a-password) to enable passwordless SSH using `ssh-keygen` and `ssh-copy-id`. It worked perfectly on my Debian system despite having been written for Raspberry Pi OS.

## Accessing Ollama

Accessing `ollama` on the server itself is trivial. Simply run:
```
curl http://localhost:11434/api/generate -d '{
  "model": "llama2",
  "prompt":"Why is the sky blue?"
}'
```
> Replace `llama2` with your preferred model.

Assuming the `OLLAMA_HOST` environment variable has been set to `0.0.0.0`, accessing `ollama` from anywhere on the network is still trivial! Simply replace `localhost` with your server's IP.

Refer to [Ollama's REST API docs](https://github.com/ollama/ollama/blob/main/docs/api.md) for more information on the entire API.

## Troubleshooting

- Disable Secure Boot in the BIOS if you're having trouble with the Nvidia drivers not working. For me, all packages were at the latest versions and `nvidia-detect` was able to find my GPU correctly, but `nvidia-smi` kept returning the `NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver` error. [Disabling Secure Boot](https://askubuntu.com/a/927470) fixed this for me. Better practice than disabling Secure Boot is to sign the Nvidia drivers yourself but I didn't want to go through that process for a non-critical server that can afford to have Secure Boot disabled.
- If you receive the `Failed to open "/etc/systemd/system/ollama.service.d/.#override.confb927ee3c846beff8": Permission denied` error from Ollama after running `systemctl edit ollama.service`, simply creating the file works to eliminate it. Use the following steps to edit the file. 
  - Run:
    ```
    sudo mkdir -p /etc/systemd/system/ollama.service.d
    sudo nano /etc/systemd/system/ollama.service.d/override.conf
    ```
  - Retry the remaining steps.
- If you still can't connect to your API endpoint, check your firewall settings. [This guide to UFW (Uncomplicated Firewall) on Debian](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-debian-10) is a good resource.

## Notes

- This is my first foray into setting up a server and ever working with Linux so there may be better ways to do some of the steps. I will update this repository as I learn more.
- I chose Debian because it is, apparently, one of the most stable Linux distros. I also went with an XFCE desktop environment because it is lightweight and I wasn't yet comfortable going full command line.
- The power draw of my EVGA FTW3 Ultra RTX 3090 was 350W at stock settings. I set the power limit to 250W and the performance decrease was negligible for my use case, which is primarily code completion in VS Code and the Q&A via chat.
- Use a user for auto-login, don't log in as root.

## References

Downloading Nvidia drivers: 
- https://wiki.debian.org/NvidiaGraphicsDrivers

Secure Boot:
- https://askubuntu.com/a/927470

Monitoring GPU usage, power draw: 
- https://unix.stackexchange.com/questions/38560/gpu-usage-monitoring-cuda/78203#78203

Passwordless `sudo`:
- https://stackoverflow.com/questions/25215604/use-sudo-without-password-inside-a-script
- https://www.reddit.com/r/Fedora/comments/11lh9nn/set_nvidia_gpu_power_and_temp_limit_on_boot/
- https://askubuntu.com/questions/100051/why-is-sudoers-nopasswd-option-not-working

Auto-login:
- https://forums.debian.net/viewtopic.php?t=149849
- https://wiki.archlinux.org/title/LightDM#Enabling_autologin

Expose Ollama to LAN:
- https://github.com/ollama/ollama/blob/main/docs/faq.md#setting-environment-variables-on-linux
- https://github.com/ollama/ollama/issues/703

Firewall:
- https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-debian-10

Passwordless `ssh`:
- https://www.raspberrypi.com/documentation/computers/remote-access.html#configure-ssh-without-a-password

Docs:

https://github.com/ollama/ollama/blob/main/docs/api.md