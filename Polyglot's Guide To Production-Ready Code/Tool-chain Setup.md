## What is a Toolchain?

A **toolchain** in software development is a set of programming tools used to perform a complex software development task or to create a software product. For compiled languages like Haskell and Rust, this typically includes:

* **Compiler:** Translates source code into machine code or an intermediate bytecode (e.g., GHC for Haskell, `rustc` for Rust).
* **Build System/Manager:** Automates the process of compiling source code, managing dependencies, linking libraries, and producing executables or libraries (e.g., Cabal/Stack for Haskell, Cargo for Rust).
* **Package Manager:** Helps in downloading, installing, and managing external libraries (dependencies) that your project uses. Often integrated with the build system.
* **Language Server (Optional but common):** Provides features like autocompletion, go-to-definition, and error checking in code editors (e.g., HLS for Haskell, `rust-analyzer` for Rust).
* **Version Manager (Often separate or integrated):** Allows developers to install and switch between different versions of the compiler and other tools (e.g., GHCup for Haskell, rustup for Rust).
* **Testing Frameworks:** Tools to write and run tests.
* **Linters/Formatters:** Tools to ensure code quality and consistent style.

Effectively, a toolchain provides the essential infrastructure to go from source code to a running application or a reusable library.

---

## Installation of Toolchains (Linux)

Here we'll look at how to get the basic toolchains up and running on a Linux system.

~~~tabs
tab: Haskell (Linux)
For Haskell on Linux, **GHCup** is the most common and recommended way to install and manage the Glasgow Haskell Compiler (GHC), the Cabal build tool, Stack (another build tool), and the Haskell Language Server (HLS).

**1. Install GHCup:**
Open your terminal and run:
```bash
curl --proto '=https' --tlsv1.2 -sSf [https://get-ghcup.haskell.org](https://get-ghcup.haskell.org) | sh
```
Follow the on-screen prompts. This script will install `ghcup` itself and typically prompt you to install recommended versions of GHC, Cabal, and HLS.

**2. Install Core Components (if not done during `ghcup` setup):**

* **Install GHC (latest recommended):**
    ```bash
    ghcup install ghc recommended
    ghcup set ghc recommended
    ```
* **Install Cabal (build tool, latest):**
    ```bash
    ghcup install cabal latest
    ghcup set cabal latest
    ```
* **Install Stack (alternative build tool/project manager, latest):**
    ```bash
    ghcup install stack latest
    ghcup set stack latest
    ```
* **Install Haskell Language Server (HLS):**
    ```bash
    ghcup install hls recommended # Installs HLS compatible with your 'recommended' GHC
    ghcup set hls recommended
    ```
    After installation, `ghcup` might suggest adding its bin directory to your PATH. Ensure this is done by sourcing your shell's configuration file (e.g., `source ~/.bashrc` or `source ~/.zshrc`) or opening a new terminal.

**Verify Installation:**
```bash
ghc --version
cabal --version
stack --version
haskell-language-server-wrapper --version
```

tab: Rust (Linux)
For Rust on Linux, **rustup** is the official and recommended tool to install and manage Rust versions and associated tools like Cargo (package manager and build system).

**1. Install rustup:**
Open your terminal and run:
```bash
curl --proto '=https' --tlsv1.2 -sSf [https://sh.rustup.rs](https://sh.rustup.rs) | sh
```
Follow the on-screen prompts. It will usually ask you to proceed with the installation (option 1 is typical). This installs `rustup` and the latest stable Rust toolchain (including `rustc`, `cargo`, `rustfmt`, `clippy`).

It will also configure your PATH environment variable. You'll need to source your shell's configuration file (e.g., `source ~/.bashrc` or `source ~/.zshrc`) or open a new terminal for the changes to take effect.

**Verify Installation:**
```bash
rustc --version
cargo --version
rustup --version
```
~~~

---

## Switching Toolchain Versions (Linux)

Once installed, you'll often need to switch between different versions of compilers or toolchains for different projects.

~~~tabs
tab: Haskell (Linux)
**Using GHCup:** (Manages GHC, Cabal, Stack, HLS versions)

* **List available GHC versions:**
    ```bash
    ghcup list ghc
    ```
* **Install a specific GHC version:**
    ```bash
    ghcup install ghc <version_number>
    # Example: ghcup install ghc 9.2.8
    ```
* **Set a default GHC version system-wide:**
    ```bash
    ghcup set ghc <version_number_or_alias>
    # Example: ghcup set ghc 9.2.8
    # Or: ghcup set ghc recommended
    ```
    You can do the same for `cabal`, `stack`, and `hls`:
    ```bash
    ghcup list cabal
    ghcup install cabal <version_number>
    ghcup set cabal <version_number_or_alias>
    ```

**Using Stack (Project-specific GHC):**

Stack primarily manages GHC versions on a per-project basis through the `resolver` field in the `stack.yaml` file.

* **Specify GHC version in `stack.yaml`:**
    Edit your project's `stack.yaml` file:
    ```yaml
    resolver: lts-20.10 # Uses GHC specified by Long Term Support snapshot 20.10
    # OR for a specific GHC version:
    # resolver: ghc-9.2.8
    ```
    When you run `stack build` or other `stack` commands, Stack will automatically download and use the GHC version specified by the resolver for that project.
* **Using a different GHC temporarily with Stack:**
    You can tell stack to use a different GHC version for a single command:
    ```bash
    stack --resolver ghc-8.10.7 exec -- ghc --version
    stack --resolver lts-18.28 build
    ```

tab: Rust (Linux)
**Using rustup:** (Manages `rustc`, `cargo`, and associated components)

* **Update all installed toolchains (and rustup itself):**
    ```bash
    rustup update
    ```
* **Install a specific toolchain channel (stable, beta, nightly):**
    ```bash
    rustup toolchain install stable
    rustup toolchain install beta
    rustup toolchain install nightly
    ```
* **Install a specific version:**
    ```bash
    rustup toolchain install <version_number>
    # Example: rustup toolchain install 1.65.0
    ```
* **List installed toolchains:**
    ```bash
    rustup toolchain list
    ```
* **Set the default toolchain system-wide:**
    ```bash
    rustup default <toolchain_name>
    # Example: rustup default stable
    # rustup default nightly
    # rustup default 1.65.0
    ```
* **Override the toolchain for the current directory (project-specific):**
    This creates a `rust-toolchain.toml` file in the current directory.
    ```bash
    rustup override set <toolchain_name>
    # Example: rustup override set nightly
    # To clear the override: rustup override unset
    ```
* **Run a command with a specific toolchain temporarily:**
    This doesn't change the default or any directory override.
    ```bash
    cargo +<toolchain_name> <command>
    # Example: cargo +nightly build
    # Example: rustc +1.65.0 --version
    ```
* **Uninstall a toolchain:**
    ```bash
    rustup toolchain uninstall <toolchain_name>
    ```
~~~

---