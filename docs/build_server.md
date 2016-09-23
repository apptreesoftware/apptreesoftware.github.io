#AppTree Build Server Documentation

## Ruby

Fastlane depends on Ruby.  The build server is configured to use a specific version of `ruby 2.2.4p230` instead of the system Ruby that came preinstalled with Mac OS X.  

There is no guarantee that Fastlane will work with a newer version of Ruby.

The version of Ruby that the build server is running is configured by rbenv.  The version is set by the `RBENV_VERSION` environment variable.  

<b>Since we are not using the system Ruby, there is no need to use `sudo` when you are installing, updating, or uninstalling gems.<b>

For more information please consult the [rbenv documentation](https://github.com/rbenv/rbenv).

### Updating Cocoapods

1. Update the `Gemfile` so that it reflect the latest version of Cocoapods.
2. Run `bundle update`.
3. Test to see that Cocoapods is working correctly by checking the version by running  `pod --version`.
4. If the version is not as expected, you may have multiple versions of Cocoapods installed.  To remedy, run `gem uninstall cocoapods` and choose to uninstall the older version.
5. `Gemfile.lock` and `Gemfile` will have changed.  Commit the changes to the repository.

### Updating Cocoapods Repository

`bundle exec pod repo update`

### Updating Fastlane

The build server will probably not require Fastlane to be updated very often in order for the builds to work.  However, if an update is required, please follow these steps:

1. Update the `Gemfile` so that it reflect the latest version of Fastlane.
2. Run `bundle update`.
3. Test to see that Fastlane is working correctly by checking the version by running `fastlane --version`.
4. If the version is not as expected, you may have multiple versions of Fastlane installed.  To remedy, run `gem uninstall fastlane` and choose to uninstall the older version.
5. `Gemfile.lock` and `Gemfile` will have changed.  Commit the changes to the repository.

## Fastlane (iOS Builds)

[Fastlane Documentation](https://docs.fastlane.tools)

### Creating a new lane

1. XCode 8 gives the option to "Automatically manage signing."  This needs to be turned off.  Fastlane will be managing the signing.

2. In `fastlane/Appfile` modify to include a new lane.
```Ruby
  for_lane :build_apptree do
    team_id 'U4U2747VWL'
    app_identifier 'com.apptreesoftware.revolution'
    apple_id 'builds@apptreesoftware.com'
  end
```

3. In `fastlane/Fastfile` modify to include a new lane.  Make certain that the scheme, provisioning profile name, team id, info plist matches the values in the XCode project.
```Ruby
  lane :build_apptree do |options|
    	ENV["SCHEME"] = "AppTreeRevolution"
    	ENV["SIGH_PROVISIONING_PROFILE_NAME"] = "AppTree Revolution"
    	ENV["SIGH_TEAM_ID"] = "U4U2747VWL"
    	ENV["INFO_PLIST"] = "../Skins/AppTreeRevolution/Info.plist"
    	ENV["VERSION_NUMBER"] = (sh "./get_version_number.sh #{ENV["INFO_PLIST"]}").strip
    	ENV["BUILD_NUMBER"] = (sh "git rev-list --count HEAD").strip
    	sh "./set_build_number.sh #{ENV["INFO_PLIST"]}"
    	ENV["BUILD_NUMBER_OUTPUT_STRING"] = File.read(Dir.pwd + '/current_build')
    	puts "Now Building #{lane_context[SharedValues::LANE_NAME]}: #{ENV["BUILD_NUMBER_OUTPUT_STRING"]}"
    	build(
      		configuration: "Release",
      		scheme: ENV["SCHEME"],
      		adhoc: false,
      		export_method: "enterprise",
      		groups: ['apptree-testers']
    	)
  end
```

4. Create a file at `fastlane/.env` and populate it with the following contents.  This also needs to be done on the build server as the file is not currently checked into the repository.
```
CRASHLYTICS_API_TOKEN = <Redacted>
CRASHLYTICS_BUILD_SECRET = <Redacted>
```

5. Try a build on your local machine and see if everything is working.

### Points of Failure

#### Cocoapods

`bundle exec pod install` may fail.  If this happens make sure cocoapods is updated to the latest release version.  Check the Podfile to see if it is properly formatted.

#### Sigh

Always look for `Found 1 matching profiles(s)`.  If Sigh does not find a profile and is forced to create one, something  is wrong.  Make certain that the provisioning profile name you have specified in the Fastfile matches the one in the developer portal.

```
+-------------------------------------+--------------------------------+
|                       Summary for sigh 1.11.1                        |
+-------------------------------------+--------------------------------+
| adhoc                               | false                          |
| development                         | false                          |
| skip_install                        | false                          |
| force                               | false                          |
| app_identifier                      | com.apptreesoftware.revolution |
| username                            | builds@apptreesoftware.com     |
| team_id                             | U4U2747VWL                     |
| team_name                           | AppTree Software LLC           |
| provisioning_name                   | AppTree Revolution             |
| ignore_profiles_with_different_name | false                          |
| skip_fetch_profiles                 | false                          |
| skip_certificate_verification       | false                          |
+-------------------------------------+--------------------------------+

[07:45:44]: Starting login with user 'builds@apptreesoftware.com'
[07:45:46]: Successfully logged in
[07:45:46]: Fetching profiles...
[07:45:53]: Found 1 matching profile(s)
[07:45:53]: Downloading provisioning profile...
[07:45:54]: Successfully downloaded provisioning profile...
[07:45:54]: Installing provisioning profile...
/Users/apptree/.jenkins/jobs/iOS_AppTree/workspace/InHouse_com.apptreesoftware.revolution.mobileprovision
[07:45:54]: Setting Provisioning Profile type to 'enterprise'
```
#### Gym

There are two phases to the process, the first starts by building all of the 3rd party dependencies.  The second phase builds all of your files.

Periodically, these builds will fail for one reason or another.  Sometimes the build will succeed the second time.  Sometimes the machine requires a reboot.

Other times the build will always fail.  The best way to troubleshoot is to try and do an Archive build in XCode on the build machine and see if any errors can be debugged there.

#### Post Build

First, the build will be uploaded to Crashlytics.  If the build stops and asks for credentials, then it is missing the file in `fastlane/.env`.

Second, if it is an App Store build, it will attempt to upload to the iTunes App Store.  This process may fail for a number of reasons but the error messages are usually very descriptive.

Third, it will rename the file with the current version and build numbers and send a success message to slack.  If the build fails for any reason it will not rename the file.

If the build was successful Jenkins will take over and perform any post-build activities, such as uploading to Dropbox.

## Jenkins

### Creating a New job

The easiest way to create a new job is to go to an existing job and select "Copy Project."

The location of the source files of any job is in `~/.jenkins/jobs/<Job Name>/workspace/`


## Build Server Maintenance

### Android Builds Failing

Sometimes, Android builds will fail consistently.  This is due to the Java daemon crashing.  To fix this, reboot the machine.  

### Fan Control

smcFanControl is an installed utility that will allow you to control the fan if the build server is getting too hot.

### General Maintenance

Onyx is an installed utility that will help to clear the system caches so that the system will run better.  Once in Onyx, run the default steps in "Automation."

## Documentation

This documentation is currently being generated from Markdown and Mkdocs.  The documentation directory is on the build server at `/Users/apptree/Docs/apptreesoftware.github.io`.
