# Step by step process - Capgo Manual updates

### Create an application 
nvm use 20

```
ng new capgo-updates-ota

cd to the app
npm install @capacitor/core @capacitor/cli @capacitor/android @capgo/capacitor-updater
npm install @capacitor/app @capacitor/splash-screen     
```
### Now, add capacitor config 

```
npx cap init over-the-air-app com.example.overtheair --web-dir=dist/over-the-air-app/browser
```
### Add a capacitor.config.ts 

Make sure to point webDir to the place where the app is built. ( ng build) 

```js
import type { CapacitorConfig } from '@capacitor/cli';

const config: CapacitorConfig = {
  appId: 'com.example.overtheair',
  appName: 'over-the-air-app',
  webDir: 'dist/capgo-updates-ota/browser', 
  server: {
    androidScheme: "https",
  },
  plugins: {
    CapacitorUpdater: {
      version: "1.0.0", 
      autoUpdate: false,
    },
  },
};

export default config;

```
Make minimal HTML + CSS changes just to have some changes. 

```js
<button (click)="checkForUpdate()">
  Check for Updates
</button>

// app.config.ts
import { provideHttpClient } from '@angular/common/http';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [provideZoneChangeDetection({ eventCoalescing: true }), provideRouter(routes), provideHttpClient()]
};

// update.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { App } from '@capacitor/app';
import { SplashScreen } from '@capacitor/splash-screen';
import { CapacitorUpdater } from '@capgo/capacitor-updater';

export interface UpdateInfo {
  version: string;
  url: string;
}

@Injectable({
  providedIn: 'root'
})
export class UpdateService {
  private readonly CURRENT_VERSION = '0.0.0';  // Initial app version
  private readonly CONFIG_URL = '';

  constructor(private http: HttpClient) {
    this.initialize();
  }

  private async initialize() {
    try {
      await CapacitorUpdater.notifyAppReady();
      console.log('[UpdateService] notifyAppReady() App ready for updates');
    } catch (error) {
      console.error('[UpdateService] Failed to initialize:', error);
    }
  }
}
```
**VERY IMPORTANT**

Now, do a build `ng build` , let it generate a build

In command line, do a capgo build. 
```
npx @capgo/cli bundle zip --path "./dist/capgo-updates-ota/browser" --name "com.example.overtheair_V1.zip"
```
It creates a zip file at root. 

```
capgo-updates-ota/
capgo-configs/
   updates/
      put zip here <--
   capacitor.config.json
```
and the json file points to a version, and the target url of the zip 

```js
{
  "version": "1.0.0",
  "url": "https://raw.githubusercontent.com/harikrishkk/capgo-configs/main/updates/com.example.overtheair_V1.zip"
}
```
Update the service 

```js
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { App } from '@capacitor/app';
import { SplashScreen } from '@capacitor/splash-screen';
import { CapacitorUpdater } from '@capgo/capacitor-updater';

export interface UpdateInfo {
  version: string;
  url: string;
}

@Injectable({
  providedIn: 'root'
})
export class UpdateService {
  private readonly CURRENT_VERSION = '0.0.0';  // Initial app version
  private readonly CONFIG_URL = 'https://raw.githubusercontent.com/harikrishkk/capgo-configs/main/capacitor.config.json';

  constructor(private http: HttpClient) {
    this.initialize();
  }

  private async initialize() {
    try {
      await CapacitorUpdater.notifyAppReady();
      console.log('[UpdateService] notifyAppReady() App ready for updates');
    } catch (error) {
      console.error('[UpdateService] Failed to initialize:', error);
    }
  }

  async checkForUpdate(): Promise<{ hasUpdate: boolean; updateInfo?: UpdateInfo }> {
    try {
      console.log('[UpdateService] Current version:', this.CURRENT_VERSION);

      // Get latest version info from GitHub
      const response = await fetch(this.CONFIG_URL);
      if (!response.ok) {
        throw new Error(`Failed to fetch version info: ${response.statusText}`);
      }
      const updateInfo = await response.json() as UpdateInfo;
      console.log('[UpdateService] Update info from GitHub:', updateInfo);

      return {
        hasUpdate: this.CURRENT_VERSION !== updateInfo.version,
        updateInfo
      };
    } catch (error) {
      console.error('[UpdateService] Update check failed:', error);
      return { hasUpdate: false };
    }
  }

  async downloadAndApplyUpdate(updateInfo: UpdateInfo): Promise<boolean> {
    try {
      // First download the bundle
      console.log('[UpdateService] Downloading update:', updateInfo.version, updateInfo.url);
      const bundle = await CapacitorUpdater.download({
        version: updateInfo.version,
        url: updateInfo.url
      });
      console.log('[UpdateService] Update downloaded, applying:', bundle);

      await SplashScreen.show();
      console.log('[UpdateService] Splash screen shown');

      await CapacitorUpdater.set(bundle);
      console.log('[UpdateService] Bundle set');

      await App.exitApp();
      console.log('[UpdateService] App exit requested');



      return true;
    } catch (error) {
      console.error('[UpdateService] Update failed:', error);
      // Hide splash screen if something goes wrong
      await SplashScreen.hide();
      return false;
    }
  }
}
```
and use the service in the component

```js
export class AppComponent {
  title = 'capgo-updates-ota';
  constructor(private updateService: UpdateService) {

  }
  async checkForUpdate() {
    const { hasUpdate, updateInfo } = await this.updateService.checkForUpdate();
    if (hasUpdate) {
      await this.updateService.downloadAndApplyUpdate(updateInfo!);
    }
  }
}
```
Now, add the android 

```
npx cap add android
ng build
npx cap sync
npx cap open android 
```

### App when loads 

![image](https://github.com/user-attachments/assets/117c7099-0be0-4f21-ad9e-ff853a19cce7)

### When we click on check for updates

![image](https://github.com/user-attachments/assets/e62cdf60-dbdc-4a6d-a28f-c199a0ce75c9)

## Automation

Add a bash file to automate the process for CI/CD

```
#!/bin/bash

# Defaults
BUILD_PATH="./dist/capgo-updates-ota/browser"
CONFIGS_ROOT="../capgo-configs"
UPDATES_FOLDER="$CONFIGS_ROOT/updates"
APP_ID="com.example.overtheair"

# Parse arguments
while [[ $# -gt 0 ]]; do
  key="$1"
  case $key in
    --version)
      VERSION="$2"
      shift
      shift
      ;;
    *)
      echo "Unknown option $1"
      exit 1
      ;;
  esac
done

if [ -z "$VERSION" ]; then
  echo "❌ Please provide a version with --version, e.g., ./bundle-update.sh --version 1.0.0"
  exit 1
fi

ZIP_NAME="${APP_ID}_${VERSION}.zip"

# 1. Check if build folder exists
if [ ! -d "$BUILD_PATH" ]; then
  echo "❌ Build folder '$BUILD_PATH' does not exist. Please run 'ng build' first."
  exit 1
fi

# 2. Run Capgo CLI to create the bundle
echo "🔄 Bundling with Capgo CLI..."
npx @capgo/cli bundle zip --path "$BUILD_PATH" --name "$ZIP_NAME"
if [ $? -ne 0 ]; then
  echo "❌ Capgo CLI bundling failed."
  exit 1
fi

# 3. Ensure updates folder exists
mkdir -p "$UPDATES_FOLDER"

# 4. Move zip to updates folder
echo "🚚 Moving bundle to $UPDATES_FOLDER"
mv "$ZIP_NAME" "$UPDATES_FOLDER/"

# 5. Delete old bundles in updates folder (keep only the new one)
echo "🧹 Deleting old bundles in $UPDATES_FOLDER"
find "$UPDATES_FOLDER" -type f -name "${APP_ID}_*.zip" ! -name "$ZIP_NAME" -delete

# 6. Update capacitor.config.json with the new URL
CONFIG_FILE="$CONFIGS_ROOT/capacitor.config.json"
RAW_URL="https://raw.githubusercontent.com/harikrishhkk/capgo-configs/main/updates/$ZIP_NAME"

echo "🔄 Updating $CONFIG_FILE with new bundle URL"
echo "\"$RAW_URL\"" > "$CONFIG_FILE"

echo "✅ capacitor.config.json updated with:"
echo "$RAW_URL"

```
and to give proper permissions to the bash file 

```
chmod +x bundle-update.sh

// now execute the command
./bundle-update.sh --version 1.0.0
```
