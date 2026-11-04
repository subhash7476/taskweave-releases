# Install TaskWeave

Verified with TaskWeave 0.2.0rc1, tunnel-client: not used, on 2026-11-03

## Goal and time

Install TaskWeave into its own virtual environment and open the control center. About 10 minutes after the prerequisites in [Before you install](01-before-you-install.md).

## You need

- Nothing besides Claude Code. Without Python 3.10 or newer, the installer sets up a private copy.
- A folder with a plain ASCII path to extract the download into.
- The [TaskWeave releases page](https://github.com/subhash7476/taskweave-releases/releases/latest) open in a browser.
- About 25 to 50 MB of download. No GitHub account or GitHub CLI.

## Steps

1. Open the [releases page](https://github.com/subhash7476/taskweave-releases/releases/latest); it shows the newest release. A release candidate is marked **Pre-release**; use one only when a guide tells you to.

2. Download the **one file** for your computer. It holds everything the install needs:

   | Your computer | Download |
   |---|---|
   | Most Windows PCs | `TaskWeave-<version>-windows-x64.zip` |
   | Windows on ARM (for example Snapdragon laptops) | No zip in 0.2.0 yet (a dependency has no Windows ARM64 build). The Windows x64 zip may work under Windows' x64 emulation, but this has not been tested. |
   | Mac with Apple silicon (M1 or newer) | `TaskWeave-<version>-macos-arm64.zip` |
   | Mac with an Intel processor | `TaskWeave-<version>-macos-x64.zip` |
   | Linux PC | `TaskWeave-<version>-linux-x64.zip` (`linux-arm64` on ARM) |

   Not sure? On Windows, **Settings, System, About** shows the system type; on a Mac, **Apple menu, About This Mac** shows **Chip** (Apple silicon) or **Processor** (Intel).

3. Extract it. On Windows, right-click the zip, choose **Extract All**, then **Extract**; do not run the installer from inside the zip. On a Mac, double-click the zip. On Linux, run `unzip` on it. You get a folder named `TaskWeave-<version>`.

4. **Windows:** open that folder and double-click `install.cmd`. If Windows shows a protection warning, choose **More info**, then **Run anyway**. The installer shows where the licence agreement is (`EULA.md` in the same folder) and asks you to accept it: type `yes` and press Enter. Nothing else is typed.

   **macOS and Linux:** open Terminal in that folder (in Finder, right-click the folder and choose **New Terminal at Folder**) and run one line, then type `yes` for the licence agreement:

   ```bash
   sh install.sh
   ```

   A download loses the executable bit, which is why the line starts with `sh`. A double-click installer is Windows-only.

5. Read the installer output. It should say `Using the wheel found in this folder: ...`, then `SHA-256 verified`, then `Installing from the dependencies bundled in wheelhouse (no download)`, then run `taskweave doctor` and open the control center at `http://127.0.0.1:8765/#setup`.

6. Open a **new** PowerShell window (macOS and Linux: a new Terminal window) and check the command; the installer put it on your PATH:

   ```powershell
   taskweave --version
   ```

   You should see `taskweave 0.2.0rc1` (or the version you downloaded).

7. Run the installer once more with no arguments. It should reuse the virtual environment, keep your settings file, and report that the same version is already installed.

8. Other ways to get the files: download them separately from the release page (the wheel, `install.cmd`, `install.ps1` or `install.sh`, `taskweave_install.py`, `constraints.txt`, `EULA.md` and `SHA256SUMS.txt`) into one folder and run the installer there. With only the installer files, the installer downloads the rest itself and checks each file against GitHub's checksum. If you use the GitHub CLI, `gh release download` fetches the same files; to install a candidate, put its tag right after `download`, for example `gh release download v0.2.0rc3` (the tag the release page shows); `gh` has no `--tag` flag.

9. If you downloaded only the wheel, you can install it yourself after creating a virtual environment:

   ```powershell
   py -3.12 -m venv C:\TaskWeave\venv
   C:\TaskWeave\venv\Scripts\python.exe -m pip install C:\path\to\taskweave_mcp-0.2.0rc1-py3-none-any.whl
   C:\TaskWeave\venv\Scripts\taskweave.exe --version
   ```

10. If the installer stops because Python could not be set up, read the message and install Python from <https://www.python.org/downloads/>, then run the installer again.

## You should see

- `Using the wheel found in this folder: taskweave_mcp-<version>-py3-none-any.whl`
- `SHA-256 verified (...)`, `Installing from the dependencies bundled in wheelhouse (no download)` and `TaskWeave is installed.`
- The control center opens on the Setup page.
- `taskweave --version` prints the version.
- A second run changes nothing and keeps the settings file.

`SHA-256 verified` means the wheel matches the checksum file that came with it. It protects against a wrong, partial or corrupted download. It does not prove who published the release: whoever can replace the wheel can replace the checksum file next to it. The `gh` path checks the checksum GitHub reports instead.

## If it does not match

| What you see | What to do |
|---|---|
| `install.ps1 was not found in this folder` | You ran `install.cmd` from inside the zip. Close the window, right-click the zip, choose **Extract All**, and run `install.cmd` from the extracted folder. |
| `The licence agreement was not accepted` | Run `install.cmd` again and type `yes` when it asks. Nothing was installed. |
| `this window cannot ask` | The installer was started without a console it can ask in. Run `install.cmd` by double-clicking it, or add `-AcceptEula` after reading `EULA.md`. |
| `Several TaskWeave files are in this folder` | Delete the wheels you do not want, keep one, and run `install.cmd` again. |
| A note that a file `is not a TaskWeave file name` | The browser may have renamed a repeated download to `... (1).whl`. Delete it and download the wheel again. |
| `The file does not match the checksum published with the release` | Delete the wheel and `SHA256SUMS.txt`, download both again from the same release, and run `install.cmd` again. Do not install the file. |
| `Python 3.10 or newer was not found, and a private Python could not be installed` | The message names the cause, usually a blocked download. Install Python from python.org, tick **Add python.exe to PATH** on Windows, open a new window, and run the installer again. |
| `the uv download does not match its pinned SHA-256` | The download was damaged or changed on the way. Run the installer again; if it repeats, report it (guide 10). Nothing was installed. |
| `Could not download TaskWeave from GitHub` | Download the zip for your computer in a browser and run the installer from the extracted folder; it needs no download then. |
| `This file came from another computer` | For a file from the TaskWeave releases page, choose **More info**, then **Run anyway**. |
| The installer opens a folder but not the control center | Run `taskweave-control --setup` from the virtual environment; the exact command is printed at the end of the installer output. |
| macOS says `Permission denied` | Run `sh install.sh` from the folder; do not double-click the file. |

## Done when

`taskweave --version` prints the version you downloaded, and the control center's Setup page is open.

## Next

[First-run setup](03-first-run-setup.md)
