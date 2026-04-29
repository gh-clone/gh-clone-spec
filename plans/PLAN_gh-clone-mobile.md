# Plan for gh-clone-mobile

### Tier 1: Ionic/Angular Foundation & Native Shell
- [ ] Scaffold the mobile application utilizing the Ionic Framework coupled natively with the existing Angular 21 Zoneless architecture to maximize component and service code reuse.
- [ ] Integrate `@capacitor/core` and essential plugins to bridge the Angular frontend with native iOS (Swift) and Android (Kotlin) device APIs.
- [ ] Build the OAuth Web Flow utilizing the `@capacitor/browser` plugin to authenticate users securely via the Rust backend without triggering pop-up blockers.
- [ ] Implement native secure credential storage utilizing `@capacitor/preferences` combined with iOS Keychain and Android Keystore to encrypt and persist the long-lived user session token.
- [ ] Design a mobile-optimized Bottom Tab Navigation bar strictly matching iOS Human Interface Guidelines (HIG) and Android Material Design standards (Home, Notifications, Search, Profile).
- [ ] Ensure strict adherence to the Primer Design System typography and color scales, subscribing to device-level Light/Dark mode changes via the Capacitor App API.
- [ ] Implement Safe Area insets handling in CSS (`env(safe-area-inset-top)`) to ensure the UI renders correctly underneath iPhone dynamic islands and Android notches.

### Tier 2: Notifications, Push & Offline Data
- [ ] Integrate Firebase Cloud Messaging (FCM) and Apple Push Notification Service (APNs) via `@capacitor/push-notifications` to receive real-time alerts.
- [ ] Architect the Rust backend to dispatch push notification payloads to FCM/APNs simultaneously with the internal Kafka notification stream.
- [ ] Build the native "Inbox" Angular view, rendering grouped notification threads with native swipe-to-dismiss and swipe-to-save gestures.
- [ ] Map incoming Push Notification tap events to the Angular Router to instantly deep-link the user to the specific Issue or PR thread.
- [ ] Utilize SQLite via `@capacitor-community/sqlite` (or IndexedDB) locally within the Ionic app to cache repository Readme files, notification states, and recent issue lists.
- [ ] Implement offline resilience: if the device enters Airplane Mode, render cached views and queue "Star" or "Save" mutations locally for replay upon network restoration.

### Tier 3: Repository Browsing & Mobile Code Review
- [ ] Implement the Mobile Repository Header component, condensing the Watch, Fork, and Star metrics into a highly compact, thumb-accessible layout.
- [ ] Build the mobile-optimized File Tree navigator, utilizing native fluid transitions (`ion-nav`) to dive deep into nested repository directories.
- [ ] Implement the Mobile Code Viewer, supporting horizontal scrolling, native pinch-to-zoom gestures, and a floating action button (FAB) for "Copy File Path".
- [ ] Build the "Swipe-to-Review" Pull Request interface: render the split or unified diff in a full-screen, horizontally swipeable mobile view.
- [ ] Implement custom touch gestures allowing users to tap a specific line number in the diff to open a native bottom-sheet modal for line-level review comments.
- [ ] Build the "Merge PR" mobile workflow, bringing the "Squash and Merge" and "Confirm" buttons into a sticky, highly visible bottom sheet.
- [ ] Implement the "Reviewers" and "Labels" selection UI utilizing native Ionic modal overlays containing sticky search-as-you-type headers.

### Tier 4: Search, Profile & OS Integrations
- [ ] Build the global mobile search experience, focusing on instant, debounced repository and user fuzzy-matching mapped to the Rust GraphQL API.
- [ ] Implement "Recent Searches" and "Trending Repositories" views tailored specifically for smaller viewports.
- [ ] Build the mobile User Profile page, rendering the contribution graph (Activity Squares) via a horizontally scrollable, highly optimized HTML5 Canvas element.
- [ ] Implement Universal Links (iOS) and App Links (Android): ensure opening a URL like `https://domain/user/repo/pull/1` from an SMS or Email automatically launches the Ionic app directly to the correct Angular route.
- [ ] Integrate Biometric Authentication (FaceID / TouchID) via Capacitor plugins to optionally lock the app when backgrounded for corporate security compliance.
- [ ] Implement the iOS Share Extension / Android Share Intent: allow users to share a URL from Safari/Chrome directly to the GitHub Mobile App to create a new Issue.
- [ ] Ensure all native App Store/Play Store assets (splash screens, icons), bundle IDs, and privacy policy URLs are generated and compliant for final binary distribution.

### Tier 5: CI/CD Pipeline & Advanced Accessibility
- [ ] Configure `fastlane` pipelines to completely automate signing, building, and deploying the `.ipa` and `.aab` artifacts to TestFlight and the Google Play Console.
- [ ] Implement robust Accessibility (a11y) labeling: ensure all interactive SVG icons have accurate `aria-label` tags explicitly read by iOS VoiceOver and Android TalkBack.
- [ ] Utilize `@capacitor/device` to read the user's preferred locale and dynamically initialize the correct i18n translation namespaces within the Angular bootstrapping phase.
- [ ] Build internal Beta-testing gates: allow organization admins to conditionally render experimental mobile features dynamically based on remote flags fetched from the Rust backend.

### Tier 6: Mobile Enterprise Device Management (MDM)
- [ ] Integrate standard AppConfig protocols into the Ionic/Capacitor application, allowing Mobile Device Management (MDM) solutions (Intune, Jamf, Workspace ONE) to inject configuration key-value pairs remotely.
- [ ] Implement the `DisableScreenshot` MDM flag on Android and iOS natively using Capacitor plugins to prevent data exfiltration from highly sensitive enterprise repositories.
- [ ] Read the `RequireVPN` AppConfig flag: enforce a network check within the Angular app that actively halts initialization and displays a warning if the corporate VPN interface is not detected.
- [ ] Support Enterprise Single Sign-On (SSO) injection: allow the MDM to push a pre-configured Identity Provider URL so enterprise users bypass the standard public login screen entirely.
- [ ] Implement a remote wipe listener: respond to MDM payload commands by instantly destroying the local SQLite cache, securely deleting all OAuth tokens from the device Keychain, and forcefully logging the user out.
- [ ] Build the "App Protected" overlay UI: if an MDM compliance check fails (e.g., the device is detected as jailbroken/rooted), freeze the application state and display a non-dismissible compliance error screen.
