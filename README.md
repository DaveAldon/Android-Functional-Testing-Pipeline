# Android-Functional-Testing-Pipeline

The intention of this project is to outline how I've gotten functional testing to be semi-automated, and for this to be a good starting point to fully automate it in a pipeline.

To succeed, we need the following to happen:

1. Create an MR in gitlab
2. Pipeline is triggered and pulls in the branch
3. Normal pipeline actions are performed, but a separate fastlane lane is triggered specifically for functional testing
4. Pull out the results of the functional tests and display them in the pipeline. Results include
   1. The html report
   2. (optional for now) A recording of the emulator running the tests

### Host Machine Requirements

1. Openjdk - `apt-get install openjdk-8-jdk`

2. Ruby and associated dependencies
   1. `apt-get install ruby-full && apt install -y build-essential`
   2. `gem install bundler && gem install fastlane`


3. Repo with the espresso test branch. Make sure these env vars are setup as well, as the `security` CLI doesn't exist in Linux and we have to bypass this gradle step:

```bash
export CI='something'
export ANDROID_STORE_PASSWORD='something'
```

Then install the repo's dependencies:

```bash
git submodule update --init && bundle install
```
   
4. Android SDK

Here are commands I use to get this setup. For some reason, the Android command line tools have a problematic installation process that hasn't been updated in a while, so the directories are wrong and must be corrected:

```bash
export ANDROID_SDK_ROOT=/usr/lib/android-sdk

mkdir -p $ANDROID_SDK_ROOT

cd $ANDROID_SDK_ROOT

wget https://dl.google.com/android/repository/commandlinetools-linux-9123335_latest.zip

unzip commandlinetools-linux-9123335_latest.zip -d $ANDROID_SDK_ROOT/cmdline-tools

mv $ANDROID_SDK_ROOT/cmdline-tools/cmdline-tools $ANDROID_SDK_ROOT/cmdline-tools/tools

rm*tools*linux*.zip
```

Update the path:

`PATH=$PATH:$ANDROID_SDK_ROOT/cmdline-tools/latest/bin:$ANDROID_SDK_ROOT/cmdline-tools/tools/bin`

Accept the license agreement:

```bash
yes | $ANDROID_SDK_ROOT/cmdline-tools/tools/bin/sdkmanager --licenses
```

Verify that the installation worked by running a debug build in the repo:

```bash
cd <REPO_DIR>

./gradlew assembleDebug
```

If it fails because of an issue finding the android sdk, you may need to use a GUI and just run the Android Studio install.

### Connect to the emulator

At this point, we can run the emulator via Google's container, and connect to it remotely.

```bash
docker run \
  -e ADBKEY="$(cat ~/.android/adbkey)" \
  --device /dev/kvm \
  --publish 8554:8554/tcp \
  --publish 5555:5555/tcp  \
  us-docker.pkg.dev/android-emulator-268719/images/30-google-x64:30.1.2
```

Once it is running, which allegedly can be verified via `adb wait-for-device` (not confirmed to be 100% accurate), we can connect to it via:

```bash
adb connect localhost:5555
```

`adb` is supposed to come with the Android SDK. If the alias isn't working, it's supposed to be in the `platform-tools` directory of the android sdk.

Verify that the `adb` connection is working, and that the device is online:

```bash
adb devices
```

This should output something like:

```bash
List of devices attached
localhost:5555 device
```

Now run the fastlane lane to run the tests:

```bash
cd <REPO_DIR>
bundle exec fastlane functionalTests
```

### Troubleshooting

If you encounter an error running the fastlane lane, we can bypass some of the build steps by running the tests directly via gradle:

```bash
./gradlew connectedDebugAndroidTest
```

If there's errors installing the build onto the emulator, we can isolate these issues by installing a debug build directly:

```bash
./gradlew assembleDebug

// This command bypasses the android security check that typically requires GUI interaction on the emulator
adb shell settings put global verifier_verify_adb_installs 0

adb install -g -r --force-queryable app/build/outputs/apk/debug/WHS-debug.apk
```
