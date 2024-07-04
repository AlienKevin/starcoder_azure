
# Set up a virtual machine

1. Go to the [azure portal](portal.azure.com) and hit "Create a resource":
![Step 0](images/0_create_a_resource.png)

2. Hit "Create" under "Virtual machine":
![Step 1](images/1_virtual_machine_create.png)

3. Give the machine a name, here we use "starcoder"
![Step 2](images/2_virtual_machine_name.png)

4. Select the security type as "Standard". Must be standard otherwise can't install Nvidia driver later. Select image as Ubuntu Server 22.04 LTS. Select size as Standard_NV36_A10_v5 - 36 vcpus.
![Step 3](images/3_virtual_machine_security_type_image_size.png)

5. Use "Password" as the Authentication type for easy access through SSH:
![Step 4](images/4_virtual_machine_authentication_type.png)

6. Create and attach a new disk to persistently store model weights and server files:
![Step 5](images/5_virtual_machine_create_and_attach_a_new_disk.png)
![Step 6](images/6_virtual_machine_create_a_new_disk_change_size.png)
![Step 7](images/7_virtual_machine_select_a_disk_size.png)

7. Hit "Review + create" to create the instance:
![Step 8](images/8_virtual_machine_review_and_create.png)

8. Once the instance is created, configure its network settings to allow inbound connections through port 8080 for use with our starcoder server:
![Step 10](images/10_virtual_machine_network_settings.png)
![Step 11](images/11_virtual_machine_create_port_rule.png)
![Step 12](images/12_virtual_machine_inbound_port_rule.png)
![Step 13](images/13_virtual_machine_add_inbound_security_rule.png)

# Install dependencies

1. Get your instance's public IP address from the instance dashboard:
![IP Address](images/9_virtual_machine_public_ip_address.png)

2. Connect via ssh
    ```
    ssh -o ServerAliveInterval=60 azureuser@<your_instance_ip_address>
    ```

3. Follow the official tutorial to attach the data disk created in the setup step. Name the mount point `/workspace` instead of `/datadir` in the tutorial. Start from the "Prepare a new empty disk" section and stop after finishing the "Verify the disk" section in the tutorial:
https://learn.microsoft.com/en-us/azure/virtual-machines/linux/attach-disk-portal

4. Grant permission to the user (otherwise always require sudo for any file operation)
    ```
    sudo chmod -R -v 777 /workspace
    ```

5. Follow the steps in the official doc to install Nvidia driver. The instance will automatically be restarted during the installation and your SSH connection will break. Reconnect to the instance after the installation finishes.
    https://learn.microsoft.com/en-us/azure/virtual-machines/extensions/hpccompute-gpu-linux#azure-portal

6. Install dependencies
    ```
    sudo apt-get update
    sudo apt-get install build-essential wget nvidia-cuda-toolkit -y
    ```
    The installation can take several minutes.

7. RESTART the instance on the instance's dashboard. Otherwise will get `Failed to initialize NVML: Driver/library version mismatch` when running `nvidia-smi`.

8. Verify that the GPU is correctly recognized:
    ```
    nvidia-smi
    ```

# Setup llama.cpp server
1. Download the a quantized version of the StarCoder 2 model from HuggingFace:
    ```
    cd /workspace
    mkdir models/
    cd models/
    wget https://huggingface.co/second-state/StarCoder2-15B-GGUF/resolve/main/starcoder2-15b-Q5_K_M.gguf
    ```
2. Build llama.cpp
    ```
    cd /workspace
    git clone https://github.com/ggerganov/llama.cpp.git
    cd llama.cpp
    CUDA_DOCKER_ARCH=compute_86 make GGML_CUDA=1
    ```

    Note: 86 is specifically for the A10 GPU. See https://github.com/distantmagic/paddler/blob/main/infra/tutorial-installing-llamacpp-aws-cuda.md#cuda-architecture-must-be-explicitly-provided

3. Start the server to listen on port 8080 for requests
    ```
    cd /workspace/llama.cpp

    ./llama-server \
        -t 10 \
        -ngl 64 \
        -b 512 \
        --ctx-size 16384 \
        -m ../models/starcoder2-15b-Q5_K_M.gguf \
        --color -c 3400 \
        --seed 42 \
        --temp 0.8 \
        --top_k 5 \
        --repeat_penalty 1.1 \
        --host :: \
        --port 8080 \
        -n -1
    ```

    Optional: Use GNU screen to allow persist access to server even when the SSH connection breaks:

    1. **Start a new screen session:**
        ```sh
        screen -S starcoder
        ```
        This starts a new screen session named `starcoder`.

    2. **Run your server inside the screen session:**
        ```sh
        ./llama-server \
            -t 10 \
            -ngl 64 \
            -b 512 \
            --ctx-size 16384 \
            -m ../models/starcoder2-15b-Q5_K_M.gguf \
            --color -c 3400 \
            --seed 42 \
            --temp 0.8 \
            --top_k 5 \
            --repeat_penalty 1.1 \
            --host :: \
            --port 8080 \
            -n -1
        ```
    This will keep running inside the screen session.

    3. **Detach from the screen session:**
        Press `Ctrl-a` followed by `d`. This detaches the screen session and keeps it running in the background.

    4. **Reattach to the screen session (if needed):**
        ```sh
        screen -r starcoder
        ```
        This reattaches you to the `starcoder` session.

    5. **List all screen sessions:**
        ```sh
        screen -ls
        ```
        This shows all the running screen sessions.

    6. **Kill a screen session (when you're done):**
        First, reattach to the session:
        ```sh
        screen -r starcoder
        ```
        Then, stop the server and exit the session:
        ```sh
        exit
        ```
