---
name: fastlane-automation
description: Automated iOS build, testing, and deployment with Fastlane. Use this skill when setting up CI/CD pipelines, automating TestFlight submissions, taking screenshots, or managing app releases. Essential for shipping reliably and reducing manual work.
compatibility: macOS, Fastlane, Xcode 14+
---

# Fastlane Automation — Build & Deploy Pipeline

**When to use:** Before first TestFlight submission. Saves hours on build and release cycles.

## Pattern: Fastlane Configuration

```bash
# iOS/fastlane/Fastfile

default_platform(:ios)

platform :ios do
  desc "Build and submit to TestFlight"
  lane :beta do
    # Increment build number
    increment_build_number(
      xcodeproj: "MyApp.xcodeproj"
    )
    
    # Build for App Store
    build_app(
      workspace: "MyApp.xcworkspace",
      scheme: "MyApp",
      configuration: "Release",
      export_method: "app-store",
      destination: "generic/platform=iOS",
      output_directory: "build",
      output_name: "MyApp.ipa",
      skip_package_ipa: false,
      skip_package_pkg: true
    )
    
    # Upload to TestFlight
    upload_to_testflight(
      ipa: "build/MyApp.ipa",
      skip_waiting_for_build_processing: false,
      skip_submission: false,
      distribute_external: false,
      notify_external_testers: true,
      changelog: "Automated beta build via Fastlane"
    )
    
    # Notify team
    slack(
      message: "TestFlight build submitted successfully!",
      slack_url: ENV["SLACK_WEBHOOK"]
    )
  end
  
  desc "Take localized screenshots"
  lane :screenshots do
    snapshot(
      scheme: "MyAppUITests",
      clean: true,
      clear_previous_screenshots: true,
      reinstall_app: true
    )
  end
  
  desc "Sync certificates and provisioning profiles"
  lane :sync_certs do
    match(
      type: "appstore",
      readonly: true,
      git_url: ENV["CERT_REPO_URL"],
      git_basic_authorization: Base64.strict_encode64("#{ENV['GIT_USER']}:#{ENV['GIT_TOKEN']}")
    )
  end
end
```

### Setup Script

```bash
#!/bin/bash
# setup-fastlane.sh

# Install Fastlane
sudo gem install fastlane -NV

# Initialize Fastlane
cd ios
fastlane init

# Configure environment
echo "SLACK_WEBHOOK=https://hooks.slack.com/..." >> .env.default
echo "CERT_REPO_URL=git@github.com:user/certs.git" >> .env.default

# Test lane
fastlane beta --verbose
```

## Implementation Checklist

- [ ] Install Fastlane: `sudo gem install fastlane -NV`
- [ ] Initialize: `fastlane init`
- [ ] Configure Apple ID and app credentials
- [ ] Set up match for certificate management
- [ ] Create beta lane with build and upload
- [ ] Configure TestFlight settings
- [ ] Add screenshot automation lane
- [ ] Set up Slack notifications
- [ ] Store secrets in environment variables
- [ ] Test full pipeline locally
- [ ] Integrate with GitHub Actions
- [ ] Document release process
