# Oracle VM VirtualBox 7.2.14 Secure Sandbox Environment

## Overview
This repository provides you with the deployment scripts and verification tools needed to build a highly secure, isolated Windows 11 sandbox. Designed specifically for Oracle VM VirtualBox 7.2.14 running on a Linux host, this setup locks down the system and strictly limits network access. 

The primary goal is to allow you to safely execute and observe software without risking your underlying host computer, ensuring the environment remains perfectly consistent for every single test you run.

## 1. Storage and Memory Containment
*   **Read-Only Hard Drive:** Your main Windows 11 virtual disk is locked so it cannot be permanently modified. Any changes made inside the sandbox are diverted to a temporary storage layer. When you shut down the machine, this temporary layer is instantly destroyed, returning your sandbox to a perfectly clean state.
*   **Direct Disk Access:** The architecture explicitly tells the hypervisor to bypass the Linux host file caching system. This ensures that disk activity inside the sandbox never interacts with your host memory buffers, reducing the risk of a storage-based breakout.
*   **Isolated Processing:** The configuration explicitly disables memory deduplication. This keeps the sandbox memory strictly walled off from your host machine.
*   **No Shared Interfaces:** All clipboard sharing, drag-and-drop features, and shared folders between your host and the sandbox are completely disabled to destroy any direct bridges between the two systems.

## 2. Firmware Security and Hardware Limits
*   **Software-Based Security Chip (TPM):** Windows 11 requires a Trusted Platform Module to run. The architecture emulates this chip entirely in software, satisfying the requirements without ever connecting the sandbox to your physical security hardware.
*   **Strict Secure Boot:** The architecture establishes a verified boot process injected with official certificates. This stops complex software from hijacking the system before the operating system even has a chance to load.
*   **Hardware Acceleration Removed:** Hardware acceleration is removed entirely for security reasons. Because of this, this setup might not be suitable for entertainment purposes such as watching YouTube videos, but it provides maximum security for your isolation needs.

## Prerequisites
*   **Host OS:** Linux.
*   **Hypervisor:** Oracle VM VirtualBox 7.2.14.
*   **Guest OS:** Windows 11 (Standard ISO).
*   **Network Drivers:** Your Windows virtual disk needs Red Hat VirtIO network drivers pre-installed so it can interface with the secure, isolated network switch.

## Deployment Strategy
You have two distinct paths for applying the configuration to the host environment. You can either execute the provided bash scripts or manually paste the configuration blocks directly into the terminal.

Automated script execution offers the most efficient deployment path on a Linux host, ensuring all configuration parameters apply simultaneously without syntax fragmentation. First, open the terminal utility and navigate to the exact folder containing the scripts. By default, standard text files lack the necessary system permissions to run as active programs. You must grant execution rights to both scripts using the change mode command by running `chmod +x deploy-sandbox.sh verify-sandbox.sh` exactly as written. Once granted, run the deployment script by calling it directly from the current directory using `./deploy-sandbox.sh` in the terminal. The prefix is mandatory because it dictates that the terminal should look in the exact folder you are currently in rather than searching the global system paths.

Manual terminal execution serves as an alternative if bypassing the scripts is necessary. You can apply the architecture manually directly through the command line interface by declaring the environment variables first to ensure the subsequent configuration commands possess the correct target context. Paste the variables mapping your machine name, disk path, storage controller, and network name directly into the terminal prompt. After defining the targets, copy the raw configuration commands from the core documentation and paste them directly into the terminal. Ensure that multi-line commands are copied and pasted entirely as a single block so the terminal processes them correctly.

## Troubleshooting Execution Errors
When executing commands in the terminal, you must keep in mind that scripts might fail to run due to mismatches, misconfigurations, network settings, and other local host variables. 

If you encounter errors during deployment, you can simply paste the error output and the script into an LLM to fix it in a second. Environment-specific syntax discrepancies or local configuration issues are straightforward to resolve with a quick prompt.

## Verification and Diagnostics
After deployment, run the provided diagnostic script. VirtualBox uses highly specific terms behind the scenes. The diagnostic tool extracts the exact state of your virtual machine for comparison against the optimal output list provided in this repository. If the outputs match, you can be certain that your sandbox is sealed and ready for use.

## Disclaimer
The author of this repository is not responsible for any damages, data loss, or system compromises caused by the use of this architecture or these scripts. You use this repository and its contents entirely at your own risk. Ensure your physical Linux host is adequately segregated from production networks before execution.