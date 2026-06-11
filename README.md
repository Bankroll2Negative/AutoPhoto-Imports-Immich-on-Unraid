# Unraid Immich Auto-Importer

A Python-based automation script designed to recursively scan folders on an **Unraid** server and upload images to a self-hosted **Immich** instance. 

This tool includes a "Dry Run" mode to verify files before uploading, duplicate detection based on filename and modification time, and rate limiting to prevent server overload.

## 🚀 Features

- **Recursive Scanning**: Automatically traverses subdirectories.
- **Dry Run Mode**: Simulate uploads without sending data to the API.
- **Duplicate Handling**: Skips files that already exist (based on `deviceAssetId`).
- **Rate Limiting**: Built-in delay between requests to protect your server.
- **Error Resilience**: Continues processing even if individual files fail.
- **Multi-format Support**: Handles JPG, PNG, HEIC, HEIF, and WEBP.

## 📋 Prerequisites

Before running the script, ensure your Unraid environment is set up correctly:

1.  **Community Applications (CA)**: Ensure the CA plugin is installed on your Unraid dashboard.
2.  **Python 3**: Installed on the Unraid host OS.
3.  **Requests Library**: The Python HTTP library required for network communication.
4.  **Immich Instance**: A running self-hosted Immich server.
5.  **API Key**: Generate a key in Immich (*Settings → API Keys*).

---

## 🛠️ Installation & Setup

### Step 1: Install Python 3 on Unraid

Unraid does not include Python by default.

1.  Open your **Unraid Dashboard**.
2.  Click on **Community Applications**.
3.  Search for `Python`.
4.  Install the **Python** plugin (by Squid) or **python3**.
5.  Wait for installation to complete.

### Step 2: Install the `requests` Library

Once Python is installed, install the necessary dependency via SSH or the Web Terminal:

```bash
pip3 install requests
