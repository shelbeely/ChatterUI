# agents.md

## Scope
Instructions for a headless Linux CI/runner to build **ChatterUI** (https://github.com/Vali-98/ChatterUI) and add/support:
1) Android SDK setup (incl. “via Android Studio” option per repo README).  
2) **APK build** using **EAS (local)** as the README requires.  
3) **OpenRouter** integration with **image generation** using `google/gemini-2.5-flash-image-preview`, with image display in-app.  
4) **OpenRouter Web Search** enablement (`:online` or `web` plugin).  
5) Use Shelbee’s Expo token and the target app id during CI where applicable.

> Repo note: your CI checks out the repository automatically, so these instructions **assume the working directory is already the ChatterUI project root** (no clone step here).

**Citations:**  
- ChatterUI README (Android section & EAS local build): Development → Android; “Building an APK”  
- OpenRouter Image Generation — images returned as base64 `data:` URL via `message.images[*].image_url.url`  
- OpenRouter Web Search — use `:online` or `plugins:[{id:'web'}]`

---

## 0) Requirements
- Node.js **18+** (LTS recommended) + npm/yarn/pnpm  
- **Java JDK 17 or 21**  
- **Android SDK** (repo asks for “install android-sdk via Android Studio”)  
- Git, curl, unzip, zip, wget, jq  
- Expo CLI + **EAS CLI** (for local APK build)  
- Environment variables (stored in CI secrets):
  - `EXPO_TOKEN= XFPWEbj3DhjwseliAPn7gDd-dSrg3w676be9l2mX`
  - `OPENROUTER_API_KEY= <your key>`
  - Optional: `OPENROUTER_HTTP_REFERER`, `OPENROUTER_X_TITLE`
- Target App Id (if your tooling references one): `275c5448-1fc2-4603-95b7-9672c6be1089`

Security: keep tokens/keys in CI secrets; **do not** commit.

---

## 1) System prep (generic Linux)
```bash
sudo apt-get update || true
sudo apt-get install -y curl wget unzip zip git ca-certificates jq

# Node via nvm (recommended)
export NVM_DIR="$HOME/.nvm"
curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
. "$NVM_DIR/nvm.sh"
nvm install --lts
node -v && npm -v

# Java (17 or 21)
sudo apt-get install -y openjdk-17-jdk || sudo apt-get install -y openjdk-21-jdk
java -version
```

---

## 2) Android SDK (two paths)

### 2A) Command-line tools only (headless friendly)
```bash
export ANDROID_SDK_ROOT="$HOME/android-sdk"
export ANDROID_HOME="$ANDROID_SDK_ROOT"
mkdir -p "$ANDROID_SDK_ROOT"

wget -O /tmp/cmdline-tools.zip https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip
unzip -q /tmp/cmdline-tools.zip -d /tmp/cli
mkdir -p "$ANDROID_SDK_ROOT/cmdline-tools"
mv /tmp/cli/cmdline-tools "$ANDROID_SDK_ROOT/cmdline-tools/latest"

# PATH (persist for the runner)
echo 'export ANDROID_SDK_ROOT="$HOME/android-sdk"' >> ~/.bashrc
echo 'export ANDROID_HOME="$ANDROID_SDK_ROOT"'     >> ~/.bashrc
echo 'export PATH="$ANDROID_SDK_ROOT/cmdline-tools/latest/bin:$ANDROID_SDK_ROOT/platform-tools:$ANDROID_SDK_ROOT/emulator:$PATH"' >> ~/.bashrc
. ~/.bashrc

yes | sdkmanager --licenses
sdkmanager "platform-tools" "platforms;android-34" "build-tools;34.0.0" "cmdline-tools;latest"
```

### 2B) “Install android-sdk via Android Studio” (as the README says)
The repo explicitly says to install the SDK via **Android Studio** for Android development. On a headless box you can run the GUI installer under a virtual display:
```bash
sudo apt-get install -y xvfb
# download + unpack Android Studio, then:
xvfb-run -a bash -lc "$HOME/android-studio/bin/studio.sh"
# In the wizard, install Android SDK + platform-tools + build-tools
```
Either 2A or 2B works; just ensure `ANDROID_SDK_ROOT` points to the installed SDK directory.

---

## 3) Dev run (per README)
From the project root (already checked out by your CI):
```bash
npm install
npx expo run:android
```

---

## 4) **Build an APK** with EAS (local)
From the README (“Building an APK”):
1. Rename `eas.json.example` → `eas.json`  
2. In `eas.json`, set `"ANDROID_SDK_ROOT"` to your SDK path (e.g., `/home/runner/android-sdk`).  
3. Run:
```bash
npm install
eas build --platform android --local
```
Tips:
- Non-interactive CI auth:
  ```bash
  export EXPO_TOKEN="XFPWEbj3DhjwseliAPn7gDd-dSrg3w676be9l2mX"
  npm i -g eas-cli expo-cli
  ```
- If your pipeline needs app id metadata, you can set it outside of the repo (some flows use `eas.json.cli.appId`).

---

## 5) Configure **OpenRouter** in ChatterUI (Remote Mode)
ChatterUI supports **Open Router** as a dedicated API in its **Remote Mode** provider list. In-app:
- Go to *Remote Mode* → Provider: **Open Router**  
- Add your `OPENROUTER_API_KEY`  
- For **Web Search**, use a model slug with `:online` (see §7) or provider plugin settings (if exposed).

> For models: select `google/gemini-2.5-flash-image-preview` for image generation. **Image outputs** are returned as base64 `data:` URLs in the chat response (see §6).

---

## 6) **Image generation** (if calling OpenRouter directly)
OpenRouter returns generated images under `choices[0].message.images[*].image_url.url` (a base64 `data:image/png;base64,...` URL). Minimal helper (only needed if you add a custom image-gen screen; the built-in provider may already handle images):

**`lib/openrouter.ts`**
```ts
export type ORImage = { type: "image_url"; image_url: { url: string } };

export async function generateImageFromPrompt(prompt: string) {
  const res = await fetch("https://openrouter.ai/api/v1/chat/completions", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.OPENROUTER_API_KEY!}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "google/gemini-2.5-flash-image-preview",
      modalities: ["image", "text"],
      messages: [{ role: "user", content: prompt }],
      stream: false,
    }),
  });

  const json = await res.json();
  const msg = json?.choices?.[0]?.message;
  const images = (msg?.images ?? []) as ORImage[];
  return images.map((i) => i.image_url.url); // data:image/png;base64,...
}
```

**Example render screen:**
```tsx
import React, { useState } from "react";
import { ScrollView, TextInput, Button, Image } from "react-native";
import { generateImageFromPrompt } from "../lib/openrouter";

export default function ImageGenScreen() {
  const [prompt, setPrompt] = useState("Witchy raccoon, red theme");
  const [images, setImages] = useState<string[]>([]);
  async function onGen(){ setImages(await generateImageFromPrompt(prompt)); }

  return (
    <ScrollView contentContainerStyle={{ padding: 16, gap: 12 }}>
      <TextInput value={prompt} onChangeText={setPrompt} placeholder="Describe your image..." />
      <Button title="Generate" onPress={onGen} />
      {images.map((uri, i) => <Image key={i} source={{ uri }} style={{ width: 320, height: 320, marginTop: 12 }} />)}
    </ScrollView>
  );
}
```

If ChatterUI’s provider already parses `message.images`, ensure the **message renderer** displays them. A generic rule: when `assistant.images?.length`, render `Image` components with those `uri`s.

---

## 7) **Web Search** (OpenRouter)
Two ways per docs:
- **Model shortcut:** append `:online`  
  ```json
  { "model": "google/gemini-2.5-flash-image-preview:online" }
  ```
- **Plugin form:**  
  ```json
  {
    "model": "openrouter/auto",
    "plugins": [{ "id": "web" }]
  }
  ```
Optional: `web_search_options` (e.g., `search_context_size`).

---

## 8) CI recipe (end‑to‑end, local EAS) — **no clone step**
```bash
#!/usr/bin/env bash
set -euo pipefail

# Working directory is already the ChatterUI project root.

# Android tools (cmdline) — or run Studio headless per §2B
export ANDROID_SDK_ROOT="$HOME/android-sdk"
export ANDROID_HOME="$ANDROID_SDK_ROOT"
export PATH="$ANDROID_SDK_ROOT/cmdline-tools/latest/bin:$ANDROID_SDK_ROOT/platform-tools:$ANDROID_SDK_ROOT/emulator:$PATH"
if [ ! -d "$ANDROID_SDK_ROOT/cmdline-tools/latest" ]; then
  mkdir -p "$ANDROID_SDK_ROOT"
  wget -O /tmp/cmdline-tools.zip https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip
  unzip -q /tmp/cmdline-tools.zip -d /tmp/cli
  mkdir -p "$ANDROID_SDK_ROOT/cmdline-tools"
  mv /tmp/cli/cmdline-tools "$ANDROID_SDK_ROOT/cmdline-tools/latest"
fi
yes | sdkmanager --licenses
sdkmanager "platform-tools" "platforms;android-34" "build-tools;34.0.0" "cmdline-tools;latest"

# Prepare EAS
mv -f eas.json.example eas.json || true
jq '.build |= with_entries(.value += {"env":{"ANDROID_SDK_ROOT":"'"$ANDROID_SDK_ROOT"'"}})' eas.json > /tmp/eas.json || true
test -s /tmp/eas.json && mv /tmp/eas.json eas.json

npm ci || npm install
npm i -g eas-cli expo-cli

# Expo auth
export EXPO_TOKEN="XFPWEbj3DhjwseliAPn7gDd-dSrg3w676be9l2mX"

# Local APK build
eas build --platform android --local
# Resulting APK/AAB path is printed by EAS; adb install that if needed.
```

---

## 9) Validation checklist
- [ ] Java `java -version` shows 17 or 21  
- [ ] `ANDROID_SDK_ROOT` set; `sdkmanager --list` works  
- [ ] `eas.json` present (renamed from example) and `ANDROID_SDK_ROOT` set there or via env  
- [ ] `eas build --platform android --local` succeeds  
- [ ] In Remote Mode, **Open Router** provider configured with API key  
- [ ] Model set to `google/gemini-2.5-flash-image-preview` (or `:online`)  
- [ ] Generated images render in the chat/message renderer

---

## 10) Troubleshooting
- **SDK not found**: verify `ANDROID_SDK_ROOT` and PATH include `cmdline-tools/latest/bin` and `platform-tools`.  
- **EAS fails keystore step locally**: EAS local uses your local Android setup; ensure Gradle caches are warm and JDK is compatible.  
- **No images in response**: ensure model supports image **generation** and request sets `modalities: ["image","text"]`.  
- **Web Search ignored**: confirm model slug ends with `:online` *or* `plugins:[{id:'web'}]`.  
- **Expo auth**: export `EXPO_TOKEN` before running EAS in CI.

