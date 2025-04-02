---
title: Comparing hybrid app frameworks
description: An exploration into options for hybrid app development targeting Android, iOS and web
slug: hybrid-app-dev
date: 2024-12-09 00:00:00+1000
image: cover.jpg
categories:
  - Development
tags:
  - React Native
  - Flutter
  - Ionic
  - .NET MAUI
weight: 1
links:
  - title: React Native
    description: Learn once, write anywhere.
    website: https://reactnative.dev/
  - title: Expo
    description: Create universal native apps with React that run on Android, iOS, and the web.
    website: https://expo.dev/
  - title: Flutter
    description: Build apps for any screen
    website: https://flutter.dev/
  - title: Ionic
    description: The mobile SDK for the Web.
    website: https://ionicframework.com/
  - title: Capacitor
    description: Cross-platform Native Runtime for Web Apps
    website: https://capacitorjs.com/docs
  - title: .NET MAUI
    description: Build native, cross-platform desktop and mobile apps all in one framework.
    website: https://dotnet.microsoft.com/en-us/apps/maui
---

I have at times felt myself wondering what the landscape of hybrid app development looks like, and after a less-than-positive experience using Ionic in the past I wanted to give myself some time to truly explore the options I had at my disposal.

The idea of this will be to write a very basic implementation of a Todo app for each of the following frameworks, providing brief findings around setup and the inner dev loop. None of the frameworks yielded a complete Todo app and this had little bearing on the findings, but it did provide a common ground to anchor any findings to.

Nothing too fancy, I've taken a "day in the life of" approach here to constrain things to a very tight timebox and taken a few notes about the experiences as I went.

{{< alert >}}
There's a bit of a brain dump following this. Jump to the [conclusion](#conclusion) if you're mostly interested in the overall findings.
{{</ alert >}}

## React Native + Expo

### General observations

- Using Expo with React Native is strongly suggested by the community based on some initial searches
- Looks like while Expo portrays itself as a paid service, doing local builds (`eas build --platform ios --local`) doesn't require their paid services
- Unsure in starting out between Expo Go and Development Build &mdash; opted for the latter, seems to be their recommendation for production workloads
- Need to sign up for a (free) Expo account if using Expo Application Services, but looks like this is unnecessary if using `npx expo run:ios`

#### Pros

- Project structure unsurprisingly looks familiar coming from a React (web) background
- Fast Refresh (i.e. hot reloading) works well
- Has a `npx @react-native-community/cli doctor` similar to Flutter which helps to diagnose any development environment issues
- [Platform-specific extensions](https://reactnative.dev/docs/platform-specific-code#platform-specific-extensions) make it look far easier to implement any platform-specific code with a consolidated import
- [EAS Updates](https://docs.expo.dev/eas-update/introduction/) looks incredibly useful however comes with a hefty price tag &mdash; this looks like it can be circumvented by setting up a custom updates server

#### Cons

- While documentation suggests that components can automatically switch between iOS and Android depending on the platform, the shipped components are very basic (a list view looked completely unstyled) and likely require a third-party UI library
- Web support seems to only be supported via a third party library, but this appears to be configured as the default when using Expo

### Platform-specific observations

#### Android

- Environment setup is a little more involved than Flutter, requiring installing JDK and manually setting `ANDROID_HOME` and `JAVA_HOME`
- Straightforward to run on both emulator or real device
- Didn't attempt to fix, but running a build using `eas build --platform android --local` failed

#### iOS

- Straightforward to get running in an emulator
- Similar to Flutter etc., running on a connected device requires a paid developer account
- Adding a list view for the todos didn't come with any styling even remotely resembling the native iOS components
- Running `eas build --platform ios --local` indicated that an active Apple account would be required to complete

#### Web

- Seems that similar to Flutter web is a second-class citizen in React Native, however Expo specifically appears to use the third-party [`react-native-web`](https://necolas.github.io/react-native-web/) library and provides the means to build and serve a web app

### Summary

The lower learning curve coming from a React background making this option hard to ignore.

Expo feels like a really nice wrapper around development, even without the paid options. It looks like using it also addresses the apparent issue around web not seemingly being supported without manually setting up a third-party library.

The apparent lack of styled components and push towards community-driven UI libraries etc. (even a basic checkbox component was marked as deprecated with a link to community alternatives) gives pause when considering React Native as a target for Android and iOS. Flutter seemed stronger in this regard, however the platform-specific extensions look like a _much_ cleaner way to go about writing platform-specific code.

While not mentioned above I did stumble across `Solito` as a library attempting to address some apparent differences in the way navigation needs to be implemented for web vs. native. I didn't get into any navigation to understand this for myself, but it did make me wonder as to the potential obstacles here.

## Flutter

### General observations

- Syntax feels more closely reminiscent of Swift UI than other web frameworks i.e. React
- Requires conditionally switching between iOS / Android / web widgets manually
- Android and iOS development seem solid, web however had some issues I was unable to resolve in a reasonable timeframe
- Ships with components, or 'widgets', implementing [both Apple's Human Interface Guidelines and Material Design](https://docs.flutter.dev/ui/widgets#design-systems)

#### Pros

- Quick to get up and running
- Useful VS Code extension
- Hot reloading works really nicely, maintains state through app changes
- Dart DevTools look like they could be incredibly helpful
- `flutter doctor` command quite useful for getting development environment set up
- Similarly, `flutter devices` and `flutter emulators` are handy - CLI tooling is quite solid overall
- Building app files is straightforward with `flutter build <target>` with `apk`, `ios`, `macos`, `web` etc.

#### Cons

- Requires learning a new language (Dart)
- Widgets aligned to platforms (Material UI vs. iOS) need to manually be implemented conditionally or rely on third-party packages for auto-switching between them
- Issues running app in debug mode for web (see [Web](#web-1) section)

### Platform-specific observations

#### Android

- Running against an emulator is pretty straightforward once Android Studio is set up
- Similarly, very easy to run against a real device once connected
- VS Code integration seemed less useful here, with no way to select the target device / emulator
- Material Design is the default, which makes it unsurprisingly very aligned to Android development

#### iOS

- VS Code extension facilitates choosing from `Simulator`, `Mac Designed for iPad`, or `macOS` as the target device
- Some permissions need to be enabled manually e.g. HTTP calls fail until adding the appropriate entitlement
- Appears that a paid developer account is required to develop locally against a real device, however actually connecting to seems relatively straightforward
- Without going too deep, the available [Cupertino widgets](https://docs.flutter.dev/ui/widgets/cupertino) look like they provide a solid ecosystem for developing complete apps aligned to the Human Interface Guidelines

#### Web

- Checking for `Platform.IsIOS` etc. breaks on web, and requires ensuring that those checks are only run _after_ checking that we're not in a web browser
- Hot reloading doesn't work here, only hot _restarting_
- VS Code not showing web / browser as a target, could be due to using a non-standard browser; needed to launch via CLI or manually created launch configuration for VS Code
- Ultimately, I had this time-boxed and was unable to actually get the web view to load &mdash; perhaps this is due to using a niche browser (Arc, which _is_ however Chromium-based) &mdash; in either case this seems to be a widely reported issue with no concrete solution

### Summary

Ultimately Flutter looks like a very promising option if we're targeting only Android and iOS. Once any issues are resolved for web, that likely becomes a valid target, however web as a target felt like a second-class citizen.

The Cupertino widgets make Flutter appear quite capable of producing a rich, native-like application, despite needing to more often than not implement separate Android and iOS widgets. Material Design as the other widget ecosystem provided by default produces a native-like Android experience.

The learning curve would be quite high owing to the need to learn a new language in Dart, especially when it comes to navigation and state management.

## Capacitor / Ionic

### General observations

- Came as a bit of a surprise upon spinning up a new project &mdash; Ionic still uses `react-scripts`

#### Pros

- Syntax is _very_ close to what a typical React web project looks like - very little learning curve and familiar workflows
- Shipped Ionic components implement both iOS and Material Design styles that automatically switch based on the platform

#### Cons

- General consensus based on some articles / conversation is that performance is not very good, especially compared to e.g. Flutter
- Community sentiment seems to be quite poor, often recommending Flutter and React Native as much better alternatives
- Documentation is a little lacking when setting up for running on other devices &mdash; requires `@capacitor/core` and `@capacitor/cli`, run `npx cap init` before we can add iOS / Android with `npx cap add iOS` etc.

### Platform-specific observations

#### Android

- Running against emulator seems to work as expected, but as with iOS was unable to get live reloading working here
- Running against a real device seemed to be on par with the emulator experience for Android

#### iOS

- Initial build worked fine, subsequent builds yielded a blank screen and no useful logs were not easy to find, assuming they exist
- Attemps to run on a real device yielded an error "Device is busy"
- While documentation suggests it is possible, could not get live reloading working when the builds did work

#### Web

- Compared to Flutter and React Native, web feels like a first-class citizen under Ionic
- Hot reloading works well for web

### Summary

Ionic feels like it has a nice happy path, but once anything goes slightly astray it starts to feel quite shaky. I would have dismissed this as the byproduct of trying to squeeze a brief intro into a single day, but this is consistent with my experience using it a couple of years prior.

The component rendering is quite nice and it's very handy just needing to render a single component and have it displayed relatively correctly per the target platform, but again this relies on that happy path. Going back to my previous experience and any custom styling etc. can be quite difficult.

Ultimately Ionic felt a fair way off the more polished experiences of Flutter and React Native. It likely has a place for a quick prototype where the majority of active development can be done in the web browser and device or emulator builds are used more sparingly. There may be another half-point here when it comes to web development specifically, but I would be more inclined to use one of the other offerings.

## .NET MAUI

### General observations

- Little confusing to start re: ability to also cover targeting web &mdash; ultimately scrapped the default MAUI project in lieu of the `.NET MAUI Blazor Hybrid and Web App` template

#### Pros

- Similar to Blazor, being able to maintain this under a single solution makes sharing data contracts etc. much more straightforward
- Looks like it's essentially Blazor from here on out, which with some experience makes me feel confident of a reasonable framework for development
- Being unable to run on iOS or Android I was unable to confirm, but the MAUI controls such as the [`ListView`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.maui.controls.listview?view=net-maui-9.0) did not appear to implement any platofrm-specific styling or functionality, appearing that this would require manual work or third-party implementations

#### Cons

- Trouble debugging via VS Code, needed to revert to running via CLI
- Running via CLI not working out of the box for iOS or Android (`dotnet build` works for all three targets)
- VS Code on macOS troubles seem to be common &mdash; seems that Rider is likely a better option

### Platform-specific observations

#### Android

- Unable to run via VS Code or CLI
- From my understanding, using the template required to support web means that the app will essentially be the web app rendered inside a web view &mdash; unsure of implications on performance or platform interactivity etc.

#### iOS

- Unable to run via VS Code or CLI
- From my understanding, using the template required to support web means that the app will essentially be the web app rendered inside a web view &mdash; unsure of implications on performance or platform interactivity etc.

#### Web

- Little confusing to start, needing to re-create project with `.NET MAUI Blazor Hybrid and Web App` template
- Started up as expected
- Hot reload worked initially, but stopped after the first update and subsequent changes did not trigger any reload

### Summary

MAUI came with the most troublesome setup, to the point that I was unable to get it running on iOS or Android emulated devices within a timebox. While this may have been environment-specific, I didn't want to consider using a specific IDE as a valid solution.

The apparent lack of out-of-the-box support for platform-specific styling also painted a less than ideal picture.

There was also some confusion over how the framework was intended to be used, with a new project based on the `.NET MAUI` template looking very different to the `.NET MAUI Blazor Hybrid App and Web App` template.

It didn't take long for me to begin feeling that this wasn't a framework I wanted to work with, despite some attraction to the idea of being able to use Blazor or a Blazor-like syntax, having used it in the past.

## Conclusion

The following is very much based around my specific environment and some aggressive timeboxes, and is reflective of my background in using predominantly React for front-end development.

| Framework    | Would use? | Setup / getting started | Tooling | Learning curve | OOTB components\* |
| ------------ | ---------- | ----------------------- | ------- | -------------- | ----------------- |
| React Native | 👍         | ⭐⭐                    | ⭐⭐⭐  | 📕             | ❌                |
| Flutter      | 👍         | ⭐⭐⭐                  | ⭐⭐⭐  | 📕 📕 📕       | ✅                |
| Ionic        | ➖         | ⭐⭐                    | ⭐      | 📕             | ✅                |
| .NET MAUI    | ❌         | ⭐                      | ⭐      | 📕 📕          | ❌                |

{{< alert >}} \* Specifically, out-of-the-box **_platform-specific_** components
{{</ alert >}}

Ultimately I would not, based on this experience, want to move forward with .NET MAUI for any of the targets.

Ionic felt a little better but this experience did little outside of reaffirming my sentiment in not wanting to use this again. I do believe however that due to the relatively pleasant happy path it provides, it may still have a place for rapidly building a simple prototype with the out-of-the-box platform-specific components. I would not pick this over Flutter or React Native for a more serious project.

React Native and Flutter felt quite even, but the deciding factors for me would be **time constraints**, specifically regarding the learning curve, and **target platforms**, looking whether we intend to target web.

For strictly iOS and Android development, it's hard to look past the out-of-the-box components that Flutter ships with. While they need to be implemented conditionally, I feel that this workflow would still be preferred to leaning on third-party components or requiring manual implementations as React Native would require.

If web was intended as a priority target alongside the two main mobile platforms, I feel that React Native creeps ahead here. There is a small caveat in the as-of-yet unexplored navigation, where the existence of the `Solito` library suggests some pain around navigation where web and native platforms are being used. This may be either addressed in recent versions, or by third-party libraries e.g. `Solito`.

I did walk away from this experience feeling that Flutter had more upside to it, but given the above considerations I would need to factor in the learning curve against the shipped platform-specific components. If pressed, I would feel compelled to choose React Native should I be starting a project today, but Flutter looks to be a very promising avenue for professional development and future usage.
