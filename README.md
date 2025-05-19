# Kivy App to *Release* APK/AAB Using GitHub Actions

## Introduction

Building a Kivy app is one thing — getting it **ready for release on Google Play** is another. Whether you want to generate a signed APK or an AAB (Android App Bundle), this guide will walk you through automating the process using **GitHub Actions**.

There are two common build types in Android development:

- **Debug Build**: For development and testing. It’s easy to install but not optimized or signed for production.
- **Release Build**: The final, polished version of your app — **signed, optimized, and ready to publish** on Google Play or distribute manually.

By the end of this guide, you’ll be able to automatically:
- Build a **release APK or AAB** from your Kivy app
- **Sign it securely** using a keystore
- Use **GitHub Actions** to automate everything in the cloud

Let’s get your app from source folder to store-ready — the fast and secure way.

🎥 **Watch the Full Tutorial on YouTube**  
👉 [Watch Now](https://youtu.be/9Nf2Oz3bPIg)

## Overview of Conversion Process

1. **Add your Kivy files to a private GitHub repository**
2. **Generate the keystore file (.jks format)**
3. **Convert the keystore to Base64**
4. **Add configurations to GitHub Secrets**
5. **Update the `buildozer.spec` file**
6. **Run the modified `build.yml` for release APK/AAB**
7. **Download and verify the signed APK/AAB**

## Step-by-Step Guide

### 1. Add Your Kivy Files to a Private GitHub Repository

1. Create a new private repository on GitHub
2. Upload your Kivy application files, including the `buildozer.spec` file, to this repository

### 2. Generate the Keystore File (.jks Format)

A keystore is required to sign your APK/AAB for release. You'll need to use the `keytool` command, which is part of the Java Development Kit (JDK).

#### Install the JDK
- Download and install the JDK from the official [Oracle website](https://www.oracle.com/java/technologies/downloads/)
- After installation, verify the installation using the command:
  ```bash
  java -version
  ```

#### Check if keytool is in the PATH
- Run the command:
  ```bash
  keytool -version
  ```
- If it's not recognized, add the JDK bin directory to your system's PATH:
  1. Press Win + R, type `sysdm.cpl`, go to Advanced → Environment Variables
  2. Under System Variables, find Path, click Edit, then New
  3. Add the JDK bin path (e.g., `C:\Program Files\Java\jdk-version.x\bin`)

#### Generate the Keystore File
1. Navigate to the location where you want to generate the keystore file
2. Run the following command (replace the parameters with your own information):
   ```bash
   keytool -genkey -v -keystore my-release-key.jks -alias my_key_alias -keyalg RSA -keysize 2048 -validity 10000 -storepass my_store_password -keypass my_key_password -dname "CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown"
   ```

#### Parameter Explanation:
- `-keystore my-release-key.jks`: Specifies the name and path of the keystore file
- `-alias my_key_alias`: The alias used to identify the key within the keystore
- `-keyalg RSA`: Specifies the algorithm used for key generation (RSA is standard)
- `-keysize 2048`: The size of the key in bits (2048 is recommended for security)
- `-validity 10000`: How many days the key will be valid (10000 days ≈ 27 years)
- `-storepass my_store_password`: The password for the keystore itself
- `-keypass my_key_password`: The password for this specific key
- `-dname "CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown"`: Distinguished name fields
  - CN: Common Name (your name or organization name)
  - OU: Organizational Unit (department)
  - O: Organization (company name)
  - L: Locality (city)
  - ST: State or Province
  - C: Country code (2-letter code, e.g., US)

**Important**: Remember the alias, keystore password, and key password. Do not lose them, as they are required for all future app updates.

### 3. Convert the Keystore to Base64

After generating the keystore file, we need to encode it into Base64 so it can be securely stored in GitHub Secrets.

1. Navigate to the directory containing your keystore file
2. Run the following command:
   ```bash
   certutil -encode my-release-key.jks temp.b64 && findstr /v /c:- temp.b64 > keystore-base64.txt && del temp.b64
   ```
3. Make sure to replace `my-release-key.jks` with the correct keystore filename
4. The `keystore_base64.txt` file will be created in the same directory

### 4. Add Configurations to GitHub Secrets

To securely store the keystore and related configurations:

1. Go to your GitHub repository
2. Navigate to Settings → Secrets and Variables → Actions
3. Add the following secrets:
   - `KEYSTORE_BASE64` - The Base64-encoded keystore content (from `keystore_base64.txt`)
   - `KEY_ALIAS` - The keystore alias (e.g., my_key_alias)
   - `KEYSTORE_PASSWORD` - The keystore password (the value you used for my_store_password)
   - `KEY_PASSWORD` - The key password (the value you used for my_key_password)

### 5. Update the buildozer.spec File

Open your `buildozer.spec` file and modify the `android.release_artifact` line:

```ini
# The format used to package the app for release mode
android.release_artifact = apk
```

For an AAB (Android App Bundle), change `android.release_artifact` to `aab`.

### 6. Run the Modified build.yml for Release APK/AAB

To automate the build process, we will modify the `build.yml` of this [GitHub Actions workflow](https://github.com/digreatbrian/buildozer-action/tree/main).

#### build.yml for APK:

```yaml
name: Build Signed APK

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-22.04

    steps:
      - uses: actions/checkout@v4

      # Step 1: Decode Keystore from base64
      - name: Decode Keystore
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d > android.jks

      # Step 2: Build Release APK using Buildozer
      - name: Build Release APK
        uses: digreatbrian/buildozer-action@v2
        with:
          python-version: 3.11
          buildozer-cmd: buildozer -v android release

      # Step 3: Sign the APK manually with .jks keystore
      - name: Sign the APK manually
        run: |
          APK_PATH=$(find ./bin -name "*-release-unsigned.apk" | head -n 1)
          jarsigner -verbose -sigalg SHA256withRSA -digestalg SHA-256 \
            -keystore android.jks \
            -storepass "${{ secrets.KEYSTORE_PASSWORD }}" \
            -keypass "${{ secrets.KEY_PASSWORD }}" \
            "$APK_PATH" "${{ secrets.KEY_ALIAS }}"

      # Step 4: Rename the signed APK
      - name: Rename the signed APK
        run: |
          APK_PATH=$(find ./bin -name "*-release-unsigned.apk" | head -n 1)
          NEW_APK_PATH="${APK_PATH/-unsigned/-signed}"
          mv "$APK_PATH" "$NEW_APK_PATH"

      # Step 5: Upload Signed APK Artifact
      - name: Upload Signed APK Artifact
        uses: actions/upload-artifact@v4
        with:
          name: signed-release-apk
          path: ./bin/*-release-signed.apk

```

#### build.yml for AAB:

For AAB, set the `android.release_artifact` setting in `buildozer.spec` to `aab`, and use the following `build.yml`:

```yaml
name: Build Signed AAB

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-22.04

    steps:
      - uses: actions/checkout@v4

      # Step 1: Decode Keystore from base64
      - name: Decode Keystore
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d > android.jks

      # Step 2: Build Release AAB using Buildozer
      - name: Build Release AAB
        uses: digreatbrian/buildozer-action@v2
        with:
          python-version: 3.11
          buildozer-cmd: buildozer -v android release

      # Step 3: Sign the AAB manually with .jks keystore
      - name: Sign the AAB manually
        run: |
          AAB_PATH=$(find ./bin -name "*-release-unsigned.aab" | head -n 1)
          jarsigner -verbose -sigalg SHA256withRSA -digestalg SHA-256 \
            -keystore android.jks \
            -storepass "${{ secrets.KEYSTORE_PASSWORD }}" \
            -keypass "${{ secrets.KEY_PASSWORD }}" \
            "$AAB_PATH" "${{ secrets.KEY_ALIAS }}"

      # Step 4: Rename the signed AAB
      - name: Rename the signed AAB
        run: |
          AAB_PATH=$(find ./bin -name "*-release-unsigned.aab" | head -n 1)
          NEW_AAB_PATH="${AAB_PATH/-unsigned/-signed}"
          mv "$AAB_PATH" "$NEW_AAB_PATH"

      # Step 5: Upload Signed AAB Artifact
      - name: Upload Signed AAB Artifact
        uses: actions/upload-artifact@v4
        with:
          name: signed-release-aab
          path: ./bin/*-release-signed.aab
```

#### Explanation of Key Steps in the `build.yml` File:

**Step 1: Decode Keystore from Base64** 

GitHub Secrets allow us to store sensitive data securely. The keystore file is stored as a Base64-encoded string in GitHub Secrets. This step decodes the keystore and saves it as `android.jks` file on the runner's filesystem.

**Step 2: Build Release APK using Buildozer** 

Uses the buildozer-action to compile your Kivy app into APK. This step will generate an **unsigned apk**(or aab) with `unsigned` word included in its name.

**Step 3: Sign the APK manually with .jks** 
   - Uses `jarsigner` to sign the unsigned APK with your keystore
   - `sigalg SHA256withRSA` specifies the signature algorithm - `SHA256withRSA` is used because:
     - It's more secure than older algorithms like MD5 or SHA1
     - Google Play now requires APKs to be signed with SHA-256 
     - It offers better protection against collision attacks
   - `digestalg SHA-256` specifies the digest algorithm, also using the more secure SHA-256 standard

This step will generate a **signed apk** (or aab), but with the `unsigned` word still in its name

**Step 4: Rename the signed APK** 

Changes the filename to indicate it's now signed (Replace `unsigned` with `signed`).

**Step 5: Upload Signed APK Artifact**

Makes the signed APK available for download from the GitHub Actions page.

### 7. Download and verify the signed APK/AAB

After the GitHub Actions workflow completes successfully, you can download the signed APK/AAB and verify that it's properly signed.

#### Download the Signed APK/AAB
1. Go to your GitHub repository
2. Click on the "Actions" tab
3. Select the most recent workflow run
4. Scroll down to the "Artifacts" section
5. Click on "signed-release-apk" or "signed-release-aab" to download the file
6. Extract the downloaded zip file to access your signed APK/AAB

#### Verifying Your APK/AAB Signature
After signing your app bundle or APK, it's crucial to verify the signature before distribution. A properly signed application is required for Google Play Store submission and ensures the integrity of your app.

To ensure that your APK is correctly signed, you can use the `jarsigner` verification tool:
1. Open a command prompt or terminal
2. Navigate to the directory containing your signed APK
3. Run the verification command:

   ```bash
   jarsigner -verify -verbose -certs your-app-name-release-signed.apk
   ```

   (Replace `your-app-name-release-signed.apk` with your actual APK filename)

4. For a successfully signed APK, you should see output similar to:

   ```
   sm     11876 Thu Apr 15 14:20:46 EDT 2025 META-INF/MANIFEST.MF
   sm     11897 Thu Apr 15 14:20:46 EDT 2025 META-INF/CERT.SF
   sm      1219 Thu Apr 15 14:20:46 EDT 2025 META-INF/CERT.RSA
           222 Thu Apr 15 14:20:20 EDT 2025 AndroidManifest.xml
           
   ...more files listed...

   s = signature was verified 
   m = entry is listed in manifest
   k = at least one certificate was found in keystore
   
   >>> Signer
   X.509, CN=Your Name/Organization, OU=Your Dept, O=Your Company, L=Your City, ST=Your State, C=US
   Signature algorithm: SHA384withRSA, 2048-bit RSA key
   [certificate is valid from YYYY-MM-DD to YYYY-MM-DD]

   - Signed by CN=Your Name/Organization, OU=Your Dept, O=Your Company, L=Your City, ST=Your State, C=US
       Digest algorithm: SHA-256
       Signature algorithm: SHA256withRSA, 2048-bit RSA key
   
   jar verified.
   ```

   The **key indicators of success** are:
   - `jar verified.` message at the end
   - `s` markers next to important files showing verified signatures
   - Certificate information matching your keystore details

### Verify AAB Signature

For Android App Bundles (AAB), use a similar command:

```bash
jarsigner -verify -verbose -certs your-app-name-release.aab
```

### Understanding Common Warnings

You may see these warnings even with properly signed apps:

```
Warning: This jar contains entries whose certificate chain is invalid.
Reason: PKIX path building failed
Warning: This jar contains entries whose signer certificate is self-signed.
Warning: This jar contains signatures that do not include a timestamp.
```

These warnings are **normal** when using **self-signed** certificates (typical for Android development) and don't indicate a problem with your release signing.

### Troubleshooting Failed Verification

If you see either of these messages, your app is NOT properly signed:
- `jar is NOT verified.`
- `jar verified, with signer errors.`

Common causes for signature failure:
- Using the wrong keystore or alias
- Incorrect keystore password
- The app was previously signed with a different key
- The signature process was interrupted
- Using old Algorithms to sign the app

## Conclusion

With the above steps, you can automate the process of building a signed release APK or AAB for your Kivy app using GitHub Actions. By securely storing your keystore in GitHub Secrets, you ensure that your build process remains both automated and secure.

## 💛 Support This Project

If you found this guide helpful and want to support more tutorials and open-source tools like this one, consider checking out my Ko-fi shop:

[![Support me on Ko-fi](./ouchentech-Sharable-Profile-Horizontal.jpg)](https://ko-fi.com/ouchentech)  

Thanks for your support 🚀 

happy coding! 🚀

---

© *OuchenTech 2025*
