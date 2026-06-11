To make this guide easily importable to GitHub, the best approach is to create a README.md file. This is the standard format for documentation on GitHub. When you upload it to a repository, GitHub automatically renders Markdown into a beautiful webpage with tables, code blocks, and syntax highlighting.Here is the content ready to be saved as README.md.How to use this:

Create a new folder on your computer.
Inside that folder, create a text file named README.md.
Paste the code block below into that file.
Save it.
Upload this folder (containing README.md and your script) to a new GitHub repository.


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

⚠️ Note: Ensure you type requests (plural). Installing request (singular) will fail.

Step 3: Create the Script

Navigate to your scripts directory via SSH:
cd /mnt/user/appdata/scripts/

Create the file:
nano immich_upload.py

Paste the Script Code (found at the bottom of this guide) into the editor.
Save (Ctrl+O, Enter) and Exit (Ctrl+X).

Step 4: Configure the Script
Edit the script to match your environment:
# At the top of immich_upload.py
API_KEY = 'YOUR_IMMICH_API_KEY'                # Replace with actual key
BASE_URL = 'http://YOUR_SERVER_IP:2283/api'     # Replace with your IP
ROOT_FOLDER = '/mnt/user/photos/import_folder'  # Replace with source path

▶️ Usage
Dry Run (Test Mode)
Always run this first to see what files would be uploaded without actually sending them.
python3 ./immich_upload.py --dry-run
Output Example:
[DRY RUN] Starting import from: /mnt/user/photos/import_folder
...
[WOULD UPLOAD] /mnt/user/photos/camera_roll/IMG_001.jpg
[WOULD UPLOAD] /mnt/user/photos/subfolder/photo.heic
...
Dry Run complete. Found 150 files. Potential uploads: 148
Active Upload
Run the script without flags to begin the actual import process.
python3 ./immich_upload.py
Output Example:
[ACTIVE MODE] Starting import from: /mnt/user/photos/import_folder
...
[OK] Uploaded: IMG_001.jpg
[SKIP] Duplicate (skipping): photo_heic.jpg
[OK] Uploaded: photo.heic
...
Scan complete. Total processed: 150

🔧 Automation with User Scripts Plugin
To run this automatically on a schedule:

Go to Plugins → User Scripts.
Click Add New Script.
Name it Immich Import.
Set Startup Type to "At scheduled times".
In the command field, enter:
python3 /mnt/user/appdata/scripts/immich_upload.py

Save and configure your schedule.


❓ Troubleshooting
ErrorSolutionModuleNotFoundError: No module named 'requests'Run pip3 install requests. Check spelling.File not foundEnsure ROOT_FOLDER in the script is an absolute path (e.g., /mnt/user/...).Connection refusedVerify BASE_URL and port (default 2283) are reachable from the Unraid host.can't find '__main__' moduleEnsure the file extension is .py, not .ph or .txt.Rate Limit ErrorsIncrease DELAY_BETWEEN_UPLOADS in the script config to 1.0 or higher.

💾 The Script
Copy the code below into your immich_upload.py file.
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

⚠️ Security Note
For production environments, avoid hardcoding your API_KEY directly in the script if the repository is public. Instead, use Environment Variables:

Modify the script to read: API_KEY = os.environ.get('IMMICH_API_KEY')
Set the variable in your terminal or User Scripts plugin before running:
export IMMICH_API_KEY="your_actual_key_here"
python3 immich_upload.py


