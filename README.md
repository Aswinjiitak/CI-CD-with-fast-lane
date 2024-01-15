# CI/CD with fast lane

### Continuous Integration (CI)
- **Integration:** The process of combining code changes from multiple contributors into a shared repository.
- **Continuous:** The practice of integrating code changes regularly, often multiple times a day.
  
In a CI setup, automated tests are run to ensure that the new code changes haven't introduced errors or conflicts with the existing codebase. This helps in identifying and fixing issues early in the developement process.

### Continuous Deployement (CD)
- **Deployement:** The process of making the application or software available for use.
- **Continuous:** The practise of automatically deploying code changes to production or staging environments as soon as they pass the CI tests.

Continuous Deployement ensures that working and tested code is rapidly deployed to production, reducing the time between writing code and making it available to users.

### Fast Lane

The term "Fast Lane" emphasizes the speed and efficiency of the CI/CD pipeline. It suggests that the CI/CD process is optimized for rapid and reliable delivery of code changes.

In a Fast Lane CI/CD setup, developers can quickly see the results of their code changes, identify and fix issues early, and deliver new features or imporvements to users more frequently. This approach is crucial in modern software development, where agility and reponsiveness to user need are higly valued. Various tools, such as Jenkins, GitLab CI/CD, Travis CI, and others, support the implementation of efficient CI/CD pipelines.

*fastlane* is the easiest way to automate beta deployements and releases for your iOS and Android apps. It handles all tedious tasks, like generating screenshots, dealing with code signing, and releasing application.

Start by creating a `Fastfile` file in repository, here's one that defines your beta or App Store release process:

```dart
lane :beta do
  increment_build_number
  build_app
  upload_to_testflight
end

lane  :release do
  capture_screenshots
  build_app
  upload_to_app_store // Upload the screenshots and the binary to iTunes
  slack               // Let your team-mates know the new version is live
end
```
Now just defined 2 different lanes, one for beta deployment, one for App Store. To release app in the App Store, all you have to do is 

```
 fastlane release
```
## Local Setup

It's recommended that to test build and deployement process locally before migrating to a cloud-based system.
You could also choose to perform continuous delivery from a local machine.

1. Install faslane `gem install fastlane` or `brew install fastlane`.
2. Create an environment variable named `FLUTTER_ROOT`, and set it to the root directory of your Flutter SDK, (This is required for the scripts that deploy for iOS.)
3. Create your Flutter project, and when ready, make sure that your project build via
   - `flutter build appbundle` and
   - `flutter build ipa`
4. Initialize the fastlane projects for each platform.
   - In your `[project]/android` directory, run `fastlane init`.
   - In your `[project]/ios` directory, run `fastlane init`.
5. Edit the `Appfiles` to ensure they have adequate metadata for your app.
   - Check that `package_name` in `[project]/android/fastlane/Appfile` matches your packages name in AndroidManifest.xml.
   - Check that app_identifier in `[project]/ios/fastlane/Appfile` also matches Info.plist’s bundle identifier. Fill in `apple_id, itc_team_id, team_id` with your respective account info.
6. Set up your local login credentials for the stores.
   - Follow the [Supply setup](https://docs.fastlane.tools/getting-started/android/setup/#setting-up-supply) steps and ensure that `fastlane supply init` successfully syncs data from your Play Store console. *Treat the .json file like your password and do not check it into any public source control repositories.*
   - Your iTunes Connect username is already in your `Appfile`’s `apple_id` field. Set the `FASTLANE_PASSWORD` shell environment variable with your iTunes Connect password. Otherwise, you’ll be prompted when uploading to iTunes/TestFlight.
7. Set up code signing
   - Follow the [Android app signing steps](https://docs.flutter.dev/deployment/android#signing-the-app).
   - On iOS, create and sign using a distribution certificate instead of a development certificate when you're ready to test and deploy using TestFlight or AppStore.
     - Create and download a distribution certificate in your [Apple Developer Account console](https://idmsa.apple.com/IDMSWebAuth/signin?appIdKey=891bd3417a7776362562d2197f89480a8547b108fd934911bcbea0110d07f757&path=%2Faccount%2Fresources%2F&rv=1).
     - `open [project]/ios/Runner.xcworkspace/` and select the distribution certificate in your target’s settings pane.
8. Create a `Fastfile` script for each platform.
   - On Android, follow the [fastlane Android beta deployement guide](https://docs.fastlane.tools/getting-started/android/beta-deployment/). Your edit could be as simple as adding a `lane` that calls `upload_to_play_store`. Set the `aab` arguments to `../build/app/outputs/bundle/release/app-release.aab` to use the app bundle `flutter build` already built.
   - On iOS, follow the [fastlane iOS beta deployment guide](https://docs.fastlane.tools/getting-started/ios/beta-deployment/). You can specify the archive path to avoid rebuilding the project. For example:
```dart
build_app(
  skip_build_archive: true,
  archive_path: "../build/ios/archive/Runner.xcarchive",
)
upload_to_testflight
```
You're now ready to perform deployments locally or migrate the deployement process to a continuous integration (CI) system.

## Running deployment locally

1. Build the release mode app
   - `flutter build appbundle`.
   - `flutter build ipa`.
2. Run the Fastfile script on each platform
   - `cd android` then `fastlane [name of the lane you created]`.
   - `cd ios` then `fastlane [name of the lane you created]`.

## Cloud build and deploy setup

First, follow the local setup section described in 'Local setup' to make sure the process works before migrating onto a cloud system like Travis.

The main thing to consider is that since cloud instances are ephermal and untrusted, you won't be leaving your credentials like you Play Store service account JSON or your iTunes distribution certificates on the server.

Continuous Integration (CI) systems generallly support encrypted environment variables to store private data. You can pass these environments variables using `--dart-define MY_VAR=MY_VALUE` while building the app.

**Tale precaution not to re-echo those variable values back onto the console in your test scripts.** Those variables are also not available in pull requests until they're merged to ensure that malicious actors cannot create a pull request that you accept and merge.

1. Make login credentials ephermal 
   - On Android:
     - Remove the `json_key_file` field from `Appfile` and store the string content of the JSON in your CI system's encrypted variable. Read the environment variable directly in your `Fastfile`.
  
     ```dart
     upload_to_play_store(
       ...
       json_key_data: ENV['<variable name>']
     )
     ```
      - Serialize your upload key (for example, using base64) and save it as an encrypted environment variable. You can deserialize it on your CI system during the install phase with
      ```dart
      echo "$PLAY_STORE_UPLOAD_KEY" | base64 --decode > [path to your upload keystore]
      ```
   - On iOS:
     - Move the local environment variable `FASTLANE_PASSWORD` to use encrypted environment variables on the CI system.
     - The CI system need access to your distribution certificate. fastlane's [Match](https://docs.fastlane.tools/actions/match/) system is recommended to synchronize your certificates across machines.
  
2. It's recommended to use a Gemfile instead of using an indeterministic `gem install fastlane` on the CI system each time to ensure the fastlane dependencies are stable and reproducible between local and cloud machines. However, this step is optional.
   
   - In both your `[project]/android` and `[project]/ios` folders, create a `Gemfile` containing the following content:
   ```
   source "https://rubygems.org"

   gem "fastlane"
   ```
   - In both directories, run `bundle update` and check both `Gemfile` and `Gemfile.lock` into source control.
   - When running locally, use `bundle exec fastlane` instead of `fastlane`.
  
3. Create the CI test script such as `.travis.yml` or `.cirrus.yml` in your repository root.
   - See [fastlane CI documentation](https://docs.fastlane.tools/best-practices/continuous-integration/) for CI specific setup.
   - Shard your script to run on both Linux and macOS platforms.
   - During the setup phase of the CI task, do the following:
     - Ensure Bundler is available using `gem install bundler`.
     - Run `bundle install` in `[project]/android` or `[project]/ios`.
     - Make sure the Flutter SDK is available and set in `PATH`.
     - For Android, ensure the Android SDK is available and the `ANDROID_SDK_ROOT` path is set.
     - For iOS, you might have to specify a dependency on Xcode (for example, `osx_image: xcode9.2`).

   - In the script phase of the CI task:
     - Run `flutter build appbundle` or `flutter build ios --release --no--codesign`, depending on the platform.
     - `cd android` or `cd ios`
     - `bundle exec fastlane [name of the lane]`

## Deploy to Beta distribution services using *fastlane* (Beta Deployment)

If you would like to distribute your beta builds to Google Play, please make sure you've done the steps from Settings up supply before continuing.

## Building your app

*fastlane* takes care of building your app by delegating to your existing Gradle build. Just add the following to `Fastfile`:

```dart
lane :beta do
  # Adjust the `build_type` and `flavor` params as needed to build the right APK for your setup
  gradle(
    task: 'assemble',
    build_type: 'Release'
  )

  # ...
end
```
Try running the lane with:

```
fastlane beta
```
When that completes you should have the appropriate APK ready to go in the standard output directory.

To get a list of all available parameters for the gradle action, run:

```
fastlane action gradle
```

## Uploading your app

After building your app, it's ready to be uploaded to a beta testing service of your choice. The beauty of *fastlane* is that you can easily switch beta providers, or even upload to mulitple at once, with a minimum of configuration. Follow that with a notification posted to the group messaging service of your choice to let the team know that you've shipped.

```dart
lane :beta do
  gradle(task: 'assemble', build_type: 'Release')
  upload_to_play_store(track: 'beta')
  slack(message: 'Successfully distributed a new beta build')
end
```
### Supported beta testing services

#### Google Play

In order to distribute to Google Play with *upload_to_play_stor*e you will need to have your Google credentials set up. Make sure you've gone through [Setting up supply](https://docs.fastlane.tools/getting-started/android/setup/#setting-up-supply) before continuing!

```dart
lane :beta do
  # ...
  gradle(
    task: 'assemble',
    build_type: 'Release'
  )
  upload_to_play_store(track: 'beta')
  # ...
end
```
To get  a list of all available options, run:

```
fastlane action upload_to_play_store
```
##### Firebase App Distribution

Install the Firebase App Distribution plugin:

```
fastlane add_plugin firebase_app_distribution
```
Authenticate with Firebase by running the `firebase_app_distribution_login` action (or using one of the other [authentication methods](https://firebase.google.com/docs/app-distribution/android/distribute-fastlane#step_2_authenticate_with_firebase)):

```
fastlane run firebase_app_distribution_login
```
Then add the `firebase_app_distribution` action to your lane:

```dart
lane :beta do
  # ...
  gradle(
    task: 'assemble',
    build_type: 'Release'
  )

  firebase_app_distribution(
    app: "1:123456789:android:abcd1234",
    groups: "qa-team, trusted-testers"
  )
  # ...
end
```
For more information and options (such as adding release notes) see the full [Getting Started](https://firebase.google.com/docs/app-distribution/android/distribute-fastlane) guide.

More information about additional supported beta testing services can be found in the [list of"Beta" actions](https://docs.fastlane.tools/actions/#beta)

## Release Notes 

Refer this [link](https://docs.fastlane.tools/getting-started/android/beta-deployment/#:~:text=of%20%22Beta%22%20actions-,Release%20Notes,-Generate%20based%20on)

## Deploy to Google Play using *fastlane* (Release Deployment)

## Building your app

*fastlane* takes care of building your app by delegating to your existing Gradle build. Just add the following to your `Fastfile`:

```dart
lane :playstore do
  gradle(
    task: 'assemble',
    build_type: 'Release'
  )
end
```
Try running the lane with:

```
fast lane playstore
```
When that completes you should have the appropriate APK ready to go in the standard output directory. To get a list of all available parameters for the `gradle` action, run:

```
fastlane action gradle
```
## Uploading your APK

To upload your binary to Google Play, *fastlane* uses a tool called *supply*. Because *supply* needs authentication information from Google, if you haven't yet done the [supply setups steps](https://docs.fastlane.tools/getting-started/android/setup/), please do those now!

With that done, simply add a call to *supply* to the lane you set up above:

```dart
lane :playstore do
  gradle(
    task: 'assemble',
    build_type: 'Release'
  )
  upload_to_play_store # Uploads the APK built in the gradle step above and releases it to all production users
end
```
This will also:

- Upload app metadata from `fastlane/metadata/android` if you previously ran `fastlane supply init`
- Upload expansion files (obbs) found under the same directory as your APK as long as:
  - They are identified by type as main or patch by containing main or patch in their file names
  - There is at most one of each type
- Upload screenshots from  `fastlane/metadata/android` if you previously ran *screengrab*
- Create a new production build
- Release the production build to all users

If you would like to capture and upload screenshots automatically as part of your deployement process, check out the [fastlane screenshots for Android](https://docs.fastlane.tools/getting-started/android/screenshots/) guide to get started!

To gradually roll out a new build you can use:

```dart
lane :playstore do
  # ...
  upload_to_play_store(
    track: 'rollout',
    rollout: '0.5'
  )
end
```
To get a list of all available parameters for the `upload_to_play_store` action, run:

```
fastlane action upload_to_play_store
```
## iOS Beta deployment using *fastlane*

## Building your app

*fastlane* takes care of building your app using an action called *build_app*, just add the following to your `Fastfile`:

```dart
lane :beta do
  build_app(scheme: "MyApp")
end
```
Additionally you can specify more options for building your app, for example

```dart
lane :beta do
  build_app(scheme: "MyApp",
            workspace: "Example.xcworkspace",
            include_bitcode: true)
end
```
Try running the lane using 

```
fastlabe beta
```
If everything works, you should have a `[ProductName].ipa` file in the current directory. To get a list of all available parameters for build_app, run `fastlane action build_app`.

#### Codesigning

Chances are that something went wrong because of code signing at the previous step. So refer this [Code Signing Guide](https://docs.fastlane.tools/codesigning/getting-started/) that helps you setting up the right code signing approach for your project.

## Uploading your app

After building your app, it's ready to be uploaded to a beta testing service of your choice. The beauty of *fastlane* is that you can easily switch beta provider, or even upload to multiple at once, without any extra work.

All you have to do is to put the name of the beta testing provider of your choice after building the app using *build_app*:

```dart
lane :beta do
  sync_code_signing(type: "appstore")    # see code signing guide for more information
  build_app(scheme: "MyApp")
  upload_to_testflight
  slack(message: "Successfully distributed a new beta build")
end
```
*fastlane* automatically passes on information about the generated `.ipa ` file from build_app to the beta testing provider of your choice.

To get a list of all available parameters for a given action, run

```
fastlane action slack
```
### Beta testing services

#### TestFlight

You can easily upload new builds to TestFlight (which is part of App Store Connect) using *fastlane*. To do so, just use the built-in testflight action after building your app

```dart
lane :beta do
  # ...
  build_app
  upload_to_testflight
end
```
Some example use cases

```dart
lane :beta do
  # ...
  build_app

  # Variant 1: Provide a changelog to your build
  upload_to_testflight(changelog: "Add rocket emoji")

  # Variant 2: Skip the "Waiting for processing" of the binary
  #   While this will speed up your build, it will not distribute
  #   the binary to your tests, nor set a changelog
  upload_to_testflight(skip_waiting_for_build_processing: true)
end
```
If you used `fastlane init` to setup *fastlane*, your Apple ID is stored in the f`astlane/Appfile`. You can also overwrite the username, using `upload_to_testflight(username: "bot@fastlane.tools")`.

To get a list of all available options, run

```
fastlane action upload_to_testflight
```
With *fastlane*, you can also automatically manage your beta testers, check out the other actions available.

#### Firebase App Distribution

Install the Firebase App Distribution plugin:

```
fastlane add_plugin firebase_app_distribution
```
Authenticate with Firebase by running the `firebase_app_distribution_login` action (or using one of the other [authentication methods](https://firebase.google.com/docs/app-distribution/ios/distribute-fastlane#step_2_authenticate_with_firebase)):

```
fastlane run firebase_app_distribution_login
```

Then add the `firebase_app_distribution` action to your lane:

```dart
lane :beta do
  # ...
  build_app

  firebase_app_distribution(
    app: "1:123456789:ios:abcd1234",
    groups: "qa-team, trusted-testers"
  )
  # ...
end
```
For more information and options (such as adding release notes) see the full [Getting Started](https://firebase.google.com/docs/app-distribution/ios/distribute-fastlane) guide.

**Other options**

#### HockeyApp
#### TestFairy

## iOS App Store deployement using *fastlane*

## Building your app

*fastlane* takes care of building your app using an action called *build_app*, just add the following to your `Fastfil`e:

```dart
lane :release do
  build_app(scheme: "MyApp")
end
```
Additionally you can specify more options for building your app, for example

```dart
lane :release do
  build_app(scheme: "MyApp",
            workspace: "Example.xcworkspace",
            include_bitcode: true)
end
```
Try running the lane using

```
fastlane release
```
If everything works, you should have a `[ProductName].ipa` file in the current directory. To get a list of all available parameters for *build_app*, run `fastlane action build_app`.

## Submitting your app

### Generating screenshots

To find out more about how to automatically generate screenshots for the App Store, check out [*fastlane* screenshots for iOS and tvOS](https://docs.fastlane.tools/getting-started/ios/screenshots/).

### Upload the binary and app metadata

After building your app, it's ready to be uploaded to the App Store. If you've already followed [iOS Beta deployment using *fastlane*](https://docs.fastlane.tools/getting-started/ios/beta-deployment/), the following code might look similar already.

```dart
lane :release do
  capture_screenshots                  # generate new screenshots for the App Store
  sync_code_signing(type: "appstore")  # see code signing guide for more information
  build_app(scheme: "MyApp")
  upload_to_app_store                  # upload your app to App Store Connect
  slack(message: "Successfully uploaded a new App Store build")
end
```
*fastlane* automatically passes on information about the generated screenshots and the binary to the `upload_to_app_store` actions of your `Fastfile`.

For a list of all options for each of the steps run `fastlane action [action_name]`.

### More details

For more details on how `upload_to_app_store` works, how you can define more options, check out [upload_to_app_store](https://docs.fastlane.tools/actions/upload_to_app_store/).

## Reference Links

- https://santhosh-adiga-u.medium.com/using-fastlane-in-flutter-to-build-ci-cd-pipeline-6238cb847d72
- https://blog.nonstopio.com/fastlane-and-flutter-integrati-f3947bd5c6aa
- https://blog.logrocket.com/fastlane-flutter-complete-guide/
- https://www.dhiwise.com/post/flutter-fastlane-streamlining-app-deployment-workflows
- https://deku.posstree.com/en/flutter/fastlane/
- https://medium.com/@kadriyemacit/fastlane-setup-for-flutter-f068376f6236
- https://semaphoreci.com/blog/automate-flutter-app-deployment-on-ios-to-testflight-using-fastlane-and-semaphore
- https://circleci.com/blog/deploy-flutter-android/
- https://dev.to/danielgomezrico/how-to-build-flutter-apps-on-ci-with-fastlane-and-reuse-some-code-2np3
- [Youtube playlist](https://youtube.com/playlist?list=PLOoOwFsn2IJOea4oB3Zt-jtnwOKjg45sE&si=UJzfdvgAsNeHqPFz)

