Of course. Here is a comprehensive guide to setting up a Python 3.13 development environment using `direnv`, `podman`, a Gentoo base image, and Jupyter notebooks.

This setup creates a seamless, containerized environment. When you `cd` into your project directory, `direnv` will automatically build and start a Podman container. Commands like `python` and `pip` will be transparently executed inside this container, giving you the power of a custom Gentoo environment with the feel of a local setup.

---

### \#\# Prerequisites

Before you begin, ensure you have the following tools installed on your host system:

- **Podman:** A daemonless container engine.
- **Direnv:** An environment switcher for your shell.

You also need to hook `direnv` into your shell. Add the following line to your `~/.bashrc`, `~/.zshrc`, or equivalent shell configuration file:

```bash
eval "$(direnv hook bash)" # Or zsh, fish, etc.
```

---

### \#\# 1. Project Structure

First, create a directory for your project. Inside this directory, we will place the configuration files and a folder for your notebooks.

```bash
mkdir my-gentoo-jupyter
cd my-gentoo-jupyter
mkdir notebooks
touch Containerfile .envrc
```

Your project structure should look like this:

```
my-gentoo-jupyter/
├── .envrc           # Direnv configuration script
├── Containerfile    # Podman container definition
└── notebooks/       # Your Jupyter notebooks will go here
```

---

### \#\# 2. The Containerfile

This file defines the Gentoo-based container. It will install Python 3.13, Pip, and Jupyter Lab. Paste the following content into your `Containerfile`.

**Note:** Gentoo compiles packages from source, so the initial build will take a significant amount of time. ക്ഷമ ধরুন (Be patient)\! 🧘

```dockerfile
# Use a recent Gentoo stage3 image as the base
FROM gentoo/stage3-amd64-systemd

# 1. Sync the package manager repository
# This fetches the latest package information.
RUN emerge-webrsync && emerge --sync

# 2. Set the Python 3.13 target and install it
# We enable the "sqlite" and "threads" USE flags for common compatibility.
RUN echo "dev-lang/python sqlite threads" >> /etc/portage/package.use/python
RUN emerge --verbose --noreplace ">=dev-lang/python-3.13"

# 3. Install Pip
RUN emerge --verbose dev-python/pip

# 4. Install Jupyter Lab and common data science libraries
RUN pip install jupyterlab pandas numpy matplotlib

# 5. Create a non-root user for better security
# We'll use UID/GID 1000, which typically matches the default user on a host system.
RUN groupadd --gid 1000 dev && \
    useradd --uid 1000 --gid 1000 --shell /bin/bash --create-home dev

# 6. Set up the working directory and switch to the new user
WORKDIR /app/notebooks
USER dev

# 7. Expose the Jupyter port
EXPOSE 8888

# 8. Define the default command to run when the container starts
# --ip=0.0.0.0 makes it accessible from the host.
# We disable the token for convenience in this local dev setup.
CMD ["jupyter", "lab", "--ip=0.0.0.0", "--port=8888", "--no-browser", "--NotebookApp.token=''"]
```

---

### \#\# 3. The `.envrc` Script

This script is the core of the automation. It tells `direnv` how to manage the Podman container and the shell environment. Paste this into your `.envrc` file.

```bash
#!/usr/bin/env bash

# --- Configuration ---
# Use the directory name for unique image and container names.
PROJECT_NAME=$(basename "$PWD")
IMAGE_NAME="${PROJECT_NAME}-img"
CONTAINER_NAME="${PROJECT_NAME}-ctr"

# --- Helper Functions ---

# A function to check if the container is running
is_container_running() {
  podman container inspect "${CONTAINER_NAME}" --format '{{.State.Running}}' 2>/dev/null | grep -q "true"
}

# A DRY (Don't Repeat Yourself) function to execute commands inside the container
in-container() {
  podman exec -it -w "/app/notebooks" "${CONTAINER_NAME}" "$@"
}


# --- Main Setup Logic ---

# Check for podman dependency
if ! has podman; then
  echo "Error: podman is not installed."
  exit 1
fi

# 1. Build the container image if it doesn't exist
if ! podman image exists "${IMAGE_NAME}"; then
  echo "Image '${IMAGE_NAME}' not found. Building..."
  podman build -t "${IMAGE_NAME}" .
  echo "Image built successfully."
fi

# 2. Start the container if it's not running
if ! is_container_running; then
  echo "Starting container '${CONTAINER_NAME}'..."
  # The --userns=keep-id flag ensures file permissions match your host user.
  # The :Z mount option handles SELinux permissions.
  podman run \
    --name "${CONTAINER_NAME}" \
    --userns=keep-id \
    -v "${PWD}/notebooks:/app/notebooks:Z" \
    -p 127.0.0.1:8888:8888 \
    -d --rm \
    "${IMAGE_NAME}"
  echo "Container started."
fi

# 3. Augment the shell environment

# Add a custom indicator to the shell prompt
export PS1="(🐍 gentoo) ${PS1}"

# Define functions that wrap commands to run them inside the container.
# Using functions is more robust than shell aliases.
python() { in-container python "$@"; }
pip() { in-container pip "$@"; }
jupyter() { in-container jupyter "$@"; }

# Export the functions so they are available in the shell
export -f python pip jupyter

# Provide a helper function to manually stop the environment's container
stop-dev-env() {
  echo "Stopping container '${CONTAINER_NAME}'..."
  podman stop "${CONTAINER_NAME}"
}

echo "✅ Gentoo + Python 3.13 environment is active."
echo "   - Jupyter Lab URL: http://localhost:8888"
echo "   - Available commands: python, pip, jupyter"
echo "   - To stop the container, run: stop-dev-env"
```

---

### \#\# 4. Activating the Environment

Now you're ready to activate the environment.

1.  Navigate to your project directory in your terminal:

    ```bash
    cd my-gentoo-jupyter
    ```

2.  The first time you enter the directory, `direnv` will show a security warning. You need to explicitly allow the script to run:

    ```bash
    direnv allow .
    ```

3.  `direnv` will now execute the `.envrc` script. It will:
    - **Build the image:** You will see the output from Podman as it builds the Gentoo image. This will take a while.
    - **Start the container:** Once the image is built, Podman will start the container in the background.
    - **Activate the environment:** Your shell prompt will change, and you'll see the success message.

Your terminal should look something like this:

```
$ cd my-gentoo-jupyter
direnv: loading ~/my-gentoo-jupyter/.envrc
Image 'my-gentoo-jupyter-img' not found. Building...
STEP 1/8: FROM gentoo/stage3-amd64-systemd
... (lots of build output) ...
Image built successfully.
Starting container 'my-gentoo-jupyter-ctr'...
... (container ID) ...
Container started.
✅ Gentoo + Python 3.13 environment is active.
   - Jupyter Lab URL: http://localhost:8888
   - Available commands: python, pip, jupyter
   - To stop the container, run: stop-dev-env
(🐍 gentoo) $
```

### \#\# 5. Using the Environment

Your environment is now active.

- **Check Python Version:** Verify that you are using Python 3.13 from the container.

  ```bash
  (🐍 gentoo) $ python --version
  Python 3.13.x
  ```

- **Access Jupyter Lab:** Open your web browser and navigate to **http://localhost:8888**. You will see the Jupyter Lab interface, and any notebook you create will be saved in the `notebooks/` directory on your host machine.

- **Install Packages:** You can install packages using `pip` directly.

  ```bash
  (🐍 gentoo) $ pip install scikit-learn
  ```

- **Leaving the Environment:** When you navigate out of the directory, `direnv` automatically unloads the environment, restoring your original shell prompt and removing the custom functions.

  ```bash
  (🐍 gentoo) $ cd ..
  direnv: unloading
  $
  ```

  The container will continue running in the background. When you `cd` back in, `direnv` will simply reconnect to it.

- **Stopping the Container:** To stop the container and free up resources, run the custom command we created:

  ```bash
  (🐍 gentoo) $ stop-dev-env
  Stopping container 'my-gentoo-jupyter-ctr'...
  ```
