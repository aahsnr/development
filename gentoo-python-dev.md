Of course. I've updated the `Containerfile` to remove the use of `make.conf` and demonstrate how to manage multiple packages with specific **USE flags** and **`accept_keywords`** using Portage's directory-based configuration.

This method gives you more granular, per-package control over your build options. The rest of the guide, including the `.envrc` file and the development workflow, remains the same.

Here is the complete, updated guide.

---

### \#\# Project Setup

First, ensure you have a project directory with the necessary files.

```bash
mkdir my-python-project
cd my-python-project
touch Containerfile .envrc
```

---

### \#\# Step 1: The Updated `Containerfile`

This revised `Containerfile` installs a broader set of development tools (`scipy`, `scikit-learn`, `git`, `neovim`, `lxml`) and configures them entirely through the `/etc/portage/` directory structure, completely avoiding `make.conf`.

Paste the following content into your `Containerfile`:

```dockerfile
# Use the latest official Gentoo stage3 image
FROM gentoo/stage3-amd64-openrc:latest

# 1. Sync the Portage tree to get the latest package definitions
RUN emerge-webrsync && emerge --sync --quiet

# 2. Configure Portage without using make.conf
#    All configurations are per-package for granular control.
RUN mkdir -p /etc/portage/package.use \
             /etc/portage/package.accept_keywords \
             /etc/portage/package.license

# 2a. Accept keywords for packages that are in testing (~amd64)
#     This is necessary for bleeding-edge versions.
RUN echo ">=dev-lang/python-3.13 ~amd64" >> /etc/portage/package.accept_keywords/python && \
    echo ">=dev-python/scikit-learn ~amd64" >> /etc/portage/package.accept_keywords/scikit-learn

# 2b. Set per-package USE flags
#     This enables specific features for each package.
#     We use separate files for better organization.
RUN echo "# Core Python features" > /etc/portage/package.use/00-python && \
    echo "dev-lang/python:3.13 threads sqlite" >> /etc/portage/package.use/00-python && \
    \
    echo "# Scientific and Data libraries" > /etc/portage/package.use/10-datascience && \
    echo "sci-libs/scipy lapack" >> /etc/portage/package.use/10-datascience && \
    echo "dev-python/pandas numpy" >> /etc/portage/package.use/10-datascience && \
    echo "dev-python/lxml threads" >> /etc/portage/package.use/10-datascience && \
    \
    echo "# Development tools" > /etc/portage/package.use/20-devtools && \
    echo "dev-vcs/git curl gpg" >> /etc/portage/package.use/20-devtools && \
    echo "app-editors/neovim python" >> /etc/portage/package.use/20-devtools

# 2c. Accept all licenses to avoid interactive prompts during emerge
RUN echo "*/* *" > /etc/portage/package.license/zz-accept-all

# 3. Install Python, Jupyter, and the new development tools
#    Using autounmask-write to automatically handle dependencies and USE flag changes.
RUN emerge --autounmask-write --verbose \
    dev-lang/python:3.13 \
    dev-python/jupyter-notebook \
    dev-python/pandas \
    dev-python/matplotlib \
    sci-libs/scipy \
    dev-python/scikit-learn \
    dev-python/lxml \
    dev-vcs/git \
    app-editors/neovim \
    app-shells/zsh && \
    etc-update --automode -5 && \
    emerge --verbose \
    dev-lang/python:3.13 \
    dev-python/jupyter-notebook \
    dev-python/pandas \
    dev-python/matplotlib \
    sci-libs/scipy \
    dev-python/scikit-learn \
    dev-python/lxml \
    dev-vcs/git \
    app-editors/neovim \
    app-shells/zsh

# 4. Create a non-root user for development
RUN useradd --create-home --shell /bin/zsh dev

# 5. Set up the working environment
WORKDIR /app
USER dev
CMD ["/bin/zsh"]
```

### Key Changes and Explanations

- **No `make.conf`**: The `RUN echo ... > /etc/portage/make.conf` line has been completely removed.
- **`package.accept_keywords`**: Instead of globally setting `ACCEPT_KEYWORDS`, we now enable testing (`~amd64`) versions for specific packages like **Python 3.13** and **scikit-learn** by adding entries to `/etc/portage/package.accept_keywords/`. This is the standard way to unmask individual packages.
- **`package.use`**: We now define all our **USE flags** in files within the `/etc/portage/package.use/` directory. This is more flexible than a single `make.conf` entry, allowing you to specify flags for exact package versions or categories. For example:
  - `dev-lang/python:3.13 threads sqlite`: Enables `threads` and `sqlite` support specifically for the Python 3.13 build.
  - `dev-vcs/git curl gpg`: Ensures Git is built with `curl` (for HTTP/S) and `gpg` (for signing) support.
- **`package.license`**: To avoid interactive prompts for license agreements, we now use `/etc/portage/package.license/` to accept all licenses, which is the modern replacement for `ACCEPT_LICENSE` in `make.conf`.
- **Expanded Package List**: The `emerge` command now installs a more complete data science and development toolchain, including **SciPy**, **scikit-learn**, **lxml**, **Git**, and **Neovim**.

---

### \#\# Step 2: The `direnv` Configuration (`.envrc`)

This file remains unchanged. It automates the process of building the image, running the container, and cleaning up afterward.

Paste the following into your `.envrc` file:

```bash
# .envrc - Direnv configuration for the Podman-based dev environment

# --- Configuration ---
IMAGE_NAME="gentoo-py13-dev"
CONTAINER_NAME="gentoo-py13-dev-container-$$" # $$ ensures a unique name per shell

# --- Helper Functions ---
image_exists() {
  podman image exists "$1"
}
container_running() {
  podman ps --filter "name=${CONTAINER_NAME}" --format '{{.Names}}' | grep -q "^${CONTAINER_NAME}$"
}

# --- Main Logic ---
# 1. Build the container image if it doesn't exist
if ! image_exists "$IMAGE_NAME"; then
  echo "--- Building container image: $IMAGE_NAME ---"
  podman build -t "$IMAGE_NAME" -f Containerfile .
fi

# 2. Start the development container if it's not already running
if ! container_running; then
  echo "--- Starting container: $CONTAINER_NAME ---"
  podman run \
    --name "$CONTAINER_NAME" \
    -d \
    -w /app \
    -v "$(pwd)":/app:z \
    -v "$HOME/.zshrc":/home/dev/.zshrc:ro \
    -v "$HOME/.config/zsh":/home/dev/.config/zsh:ro \
    -p 127.0.0.1:8888:8888 \
    "$IMAGE_NAME" \
    sleep infinity > /dev/null
fi

# 3. Create a handy alias to enter the container
alias enter-dev="podman exec -it -u dev $CONTAINER_NAME zsh"
echo "✅ Dev environment ready. Use 'enter-dev' to get a shell in the container."

# --- Cleanup Function ---
on_exit() {
  echo "--- Stopping and removing container: $CONTAINER_NAME ---"
  podman stop "$CONTAINER_NAME" > /dev/null
  podman rm "$CONTAINER_NAME" > /dev/null
  echo "🧹 Environment cleaned up."
}
add_on_exit on_exit
```

---

### \#\# Step 3: The Development Workflow 🧑‍💻

Your workflow is the same as before, but now you have more tools available inside the container.

1.  **Activate Environment**:

    ```bash
    cd my-python-project
    direnv allow
    ```

    If you already have an older image built, you may want to remove it (`podman rmi gentoo-py13-dev`) to trigger a rebuild with the new `Containerfile`.

2.  **Enter the Container**:

    ```bash
    enter-dev
    ```

3.  **Launch Jupyter Notebook**:

    ```bash
    # Inside the container
    jupyter notebook --ip=0.0.0.0 --port=8888
    ```

    Access it from your host browser using the URL provided in the terminal.

4.  **Exit and Cleanup**:

    ````bash
    # Exit the container shell with 'exit' or Ctrl+D
    # Then, leave the project directory
    cd ..
    ```direnv` will automatically stop and remove the container.
    ````
