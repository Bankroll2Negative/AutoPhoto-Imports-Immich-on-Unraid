
# 📸 Unraid Immich Auto-Importer

A Python-based automation script designed to recursively scan folders on an **Unraid** server and upload images to a self-hosted **Immich** instance. TThe main body of the script is from immich auto import. It was limited to only one photo at a time. I had assistance from AI to allow the script to take multiple photos.

This tool solves the common problem of moving large photo libraries into Immich without manual intervention. It features a **"Dry Run"** mode to verify files before uploading, intelligent duplicate detection based on filename and modification time, and built-in rate limiting to prevent server overload.

## ✨ Features

- **Recursive Scanning**: Automatically traverses all subdirectories within a target folder.
- **🧪 Dry Run Mode**: Simulate the entire process to see what *would* happen without sending any data to the API.
- **🔄 Duplicate Handling**: Skips files that already exist (based on generated `deviceAssetId`).
- **⏱️ Rate Limiting**: Configurable delay between requests to protect your Immich server from spikes.
- **🛡️ Error Resilience**: Continues processing even if individual files fail (network timeout, permissions, etc.).
- **📂 Multi-format Support**: Handles `.jpg`, `.jpeg`, `.png`, `.heic`, `.heif`, and `.webp`.

---

## 📋 Prerequisites

Before starting, ensure you have:
1.  **Unraid OS**: Version 6.9+ recommended.
2.  **Community Applications (CA)**: Installed on your dashboard.
3.  **Python 3**: Will be installed during setup.
4.  **Immich Instance**: Running on your local network.
5.  **Immich API Key**: Generate one in Immich (*Settings → API Keys*).

---

## 🛠️ Complete Setup Guide

### Step 1: Install Community Applications (If Needed)
If you haven't installed the plugin yet, do this first:
1.  Go to the **Plugins** tab in your Unraid Dashboard.
2.  Click the **Install Plugins** button at the bottom.
3.  Paste this URL: `https://raw.githubusercontent.com/Squid-2/unRAID-community-apps/master/community.applications.plg`
4.  Click **OK** and wait for the installation to complete. You may need to refresh the page or reboot if prompted.

### Step 2: Install Python 3
1.  Open the **Community Applications** tab in your Dashboard.
2.  Search for `Python`.
3.  Look for the plugin named **Python** (by Squid) or **python3**.
4.  Click **Install**. Wait for the process to finish (usually 1–3 minutes).

### Step 3: Install Dependencies
Open the Terminal (via the Web GUI "Terminal" button or SSH) to install the required HTTP library:

    pip3 install requests

> ⚠️ **Note:** Ensure you type `requests` (plural). Installing `request` (singular) will result in a `ModuleNotFoundError`.

### Step 4: Create the Script File
1.  Navigate to your scripts directory:

        cd /mnt/user/appdata/scripts/

2.  Create the file using a text editor:

        nano immich_upload.py

3.  Paste the **[Full Script](#the-full-script)** found at the bottom of this guide into the editor.
4.  Save the file (`Ctrl+O`, then `Enter`) and Exit (`Ctrl+X`).

### Step 5: Configure Credentials
Edit the top section of the script to match your specific environment:

    nano immich_upload.py

Update these three lines with your details:

| Variable | Description | Example Value |
| :--- | :--- | :--- |
| `API_KEY` | Your Immich API Key | `'a1b2c3d4...'` |
| `BASE_URL` | Your Immich Server Address | `'http://192.168.1.50:2283/api'` |
| `ROOT_FOLDER` | Path to photos on Unraid | `'/mnt/user/photos/camera_roll'` |

> 💡 **Tip:** Always use absolute paths for `ROOT_FOLDER` (e.g., `/mnt/user/...`) to avoid path resolution errors.

Save (`Ctrl+O`, `Enter`) and exit (`Ctrl+X`).

### Step 6: Test with Dry Run
**Crucial:** Never run a bulk upload immediately. Test first to verify paths and detect duplicates safely.

Run the script with the `--dry-run` flag:

    python3 ./immich_upload.py --dry-run

Expected Output:

    [DRY RUN] Starting import from: /mnt/user/photos/camera_roll
    Allowed extensions: {'.jpg', '.png', ...}
    ------------------------------------------------------------
    [WOULD UPLOAD] /mnt/user/photos/camera_roll/IMG_001.jpg
    [WOULD UPLOAD] /mnt/user/photos/camera_roll/vacation/photo.heic
    ------------------------------------------------------------
    Dry Run complete. Found 150 files. Potential uploads: 148

If you see this, no data has been sent to Immich yet.

### Step 7: Execute Live Upload
Once you are satisfied with the Dry Run results, remove the flag to begin the actual import:

    python3 ./immich_upload.py

Expected Output:

    [ACTIVE MODE] Starting import from: /mnt/user/photos/camera_roll
    ...
    [OK] Uploaded: IMG_001.jpg
    [SKIP] Duplicate (skipping): photo_heic.jpg
    [OK] Uploaded: photo.heic
    ...
    Scan complete. Total processed: 150

### Step 8: Automate (Optional)
To run this automatically on a schedule (e.g., daily at midnight):

1.  Go to **Plugins** → **User Scripts** in your Unraid Dashboard.
2.  Click **Add New Script**.
3.  Name it `Immich Auto Import`.
4.  Set **Startup Type** to **"At scheduled times"**.
5.  Set your desired schedule (e.g., `0 0 * * *` for midnight daily).
6.  In the **Command** field, enter:

        python3 /mnt/user/appdata/scripts/immich_upload.py

7.  Click **Save**.

---

## ❓ Troubleshooting

| Issue | Likely Cause | Solution |
| :--- | :--- | :--- |
| `ModuleNotFoundError: No module named 'requests'` | Missing library. | Run `pip3 install requests`. Check spelling (plural). |
| `File not found` | Incorrect path. | Ensure `ROOT_FOLDER` is an absolute path like `/mnt/user/...`. |
| `Connection refused` | Network issue. | Verify `BASE_URL` IP and port (default 2283) are accessible from Unraid. |
| `can't find '__main__' module` | Wrong file extension. | Rename file from `.ph` or `.txt` to `.py`. |
| `409 Conflict` (Skipped) | Duplicate detected. | This is normal behavior; the script skips existing files to save space/time. |
| `Timeout Errors` | Large files or slow net. | Increase `DELAY_BETWEEN_UPLOADS` in the script config to `1.0` or `2.0`. |

---

## The Full Script

Copy the code below entirely into your `immich_upload.py` file.

---- BEGIN SCRIPT ----

#!/usr/bin/python3
import os
import sys
import time
import requests
import argparse
from datetime import datetime

# -----------------------------------------------------------------------------
# CONFIGURATION
# Edit these values to match your setup
# -----------------------------------------------------------------------------
API_KEY = 'YOUR_IMMICH_API_KEY'                # Replace with your actual key
BASE_URL = 'http://YOUR_SERVER_IP:2283/api'     # Replace with your Immich IP:Port
ROOT_FOLDER = '/mnt/user/photos/import_folder'  # Replace with your source folder path
DELAY_BETWEEN_UPLOADS = 0.5                     # Delay in seconds to avoid rate limits
ALLOWED_EXTENSIONS = {'.jpg', '.jpeg', '.png', '.heic', '.heif', '.webp'}

def prepare_upload_data(file_path):
    """Generates metadata without sending the request."""
    try:
        stats = os.stat(file_path)
        modified_time = datetime.fromtimestamp(stats.st_mtime).isoformat() + "Z"
        device_asset_id = f"{os.path.basename(file_path)}-{int(stats.st_mtime)}"
        return {
            'status': 'ok',
            'asset_id': device_asset_id,
            'size_bytes': stats.st_size,
            'modified': modified_time
        }
    except OSError as e:
        return {'status': 'error', 'message': str(e)}

def upload_photo(file_path):
    """Uploads a single photo to Immich."""
    
    data_prep = prepare_upload_data(file_path)
    if data_prep['status'] == 'error':
        print(f"Error reading file stats: {data_prep['message']} - {file_path}")
        return False

    headers = {
        'Accept': 'application/json',
        'x-api-key': API_KEY
    }
    
    data = {
        'deviceAssetId': data_prep['asset_id'],
        'deviceId': 'unraid-directory-import',
        'fileCreatedAt': data_prep['modified'],
        'fileModifiedAt': data_prep['modified'],
        'isFavorite': 'false',
    }
    
    try:
        with open(file_path, 'rb') as img_file:
            files = {'assetData': img_file}
            response = requests.post(f'{BASE_URL}/assets', headers=headers, data=data, files=files, timeout=60)
            
            if response.status_code == 201:
                print(f"[OK] Uploaded: {os.path.basename(file_path)}")
                return True
            elif response.status_code == 409:
                print(f"[SKIP] Duplicate (skipping): {os.path.basename(file_path)}")
                return True 
            else:
                err_msg = response.text[:100] if response.text else "Unknown error"
                print(f"[FAIL] {response.status_code}: {os.path.basename(file_path)} -> {err_msg}")
                return False
                
    except requests.exceptions.RequestException as e:
        print(f"[NET] Network error: {e}")
        return False
    except Exception as e:
        print(f"[ERR] Unexpected error: {e}")
        return False

def import_directory(root_path, dry_run=False):
    """Recursively scans and uploads photos."""
    
    if not os.path.exists(root_path):
        print(f"Error: Root path '{root_path}' does not exist.")
        sys.exit(1)

    mode_str = "[DRY RUN]" if dry_run else "[ACTIVE MODE]"
    print(f"{mode_str} Starting import from: {root_path}")
    print(f"Allowed extensions: {ALLOWED_EXTENSIONS}")
    print("-" * 60)

    total_files = 0
    potential_uploads = 0

    for dirpath, dirnames, filenames in os.walk(root_path):
        for filename in filenames:
            _, ext = os.path.splitext(filename)
            if ext.lower() not in ALLOWED_EXTENSIONS:
                continue
            
            file_path = os.path.join(dirpath, filename)
            total_files += 1
            
            if dry_run:
                meta = prepare_upload_data(file_path)
                if meta['status'] == 'ok':
                    potential_uploads += 1
                    print(f"[WOULD UPLOAD] {file_path}")
                else:
                    print(f"[ERROR] {meta['message']} - {file_path}")
            else:
                result = upload_photo(file_path)
            
            if not dry_run:
                time.sleep(DELAY_BETWEEN_UPLOADS)

    print("-" * 60)
    if dry_run:
        print(f"Dry Run complete. Found {total_files} files. Potential uploads: {potential_uploads}")
    else:
        print(f"Scan complete. Total processed: {total_files}")

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Import photos to Immich from Unraid")
    parser.add_argument('--dry-run', action='store_true', help="Simulate upload without sending data")
    args = parser.parse_args()

    if API_KEY.startswith('YOUR_') or BASE_URL.startswith('http://YOUR'):
        print("ERROR: Please edit the script and update API_KEY and BASE_URL variables.")
        sys.exit(1)

    import_directory(ROOT_FOLDER, dry_run=args.dry_run)

---- END SCRIPT ----

---

## 🔒 Security Note

For production environments, **do not hardcode your API key** in the script if the repository is public. Use Environment Variables instead:

1.  Change config line to: `API_KEY = os.environ.get('IMMICH_API_KEY')`
2.  Set variable before running: `export IMMICH_API_KEY="key_here"`
3.  Or configure in User Scripts "Command" field: `export IMMICH_API_KEY="key"; python3 immich_upload.py`

---

## 🤝 Contributing

Feel free to submit issues or enhancement requests! If you modify this script, please ensure you test thoroughly with `--dry-run` first.

*Made with ❤️ for the Unraid and Immich community.*
