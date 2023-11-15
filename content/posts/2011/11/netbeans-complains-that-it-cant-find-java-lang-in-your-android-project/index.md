---
title: "NetBeans Complains That it Can't Find Java.Lang in your Android Project."
date: "2011-11-04"
categories: 
  - "android"
  - "linux"
  - "netbeans"
---

Using NetBeans (version 7.0.1) to develop Android applications and for no apparent reason, a project that worked yesterday won't run any more. It compiles without error but refuses to run - whether or not the emulator is running - and doesn't produce an error. Hmmm.  In the source code for the main Java class (yes, I said Java - anyone who knows me will be having a fit right now!) the very first line is flagged with a red (ok, pink!) dot. The line itself simply says:

`package uk.co.dunbarit.myprojectname;`

Rolling the mouse over the line pops up a helpful hint which states that the system _cannot find java.lang in classpath or bootclasspath_.

Now, as my work colleagues will tell you, I'm not a Java developer by any means. But this was working yesterday, so why not today?

Right-click the project name in the project tab, and select the option - near the bottom - to _rebuild broken project_. This pops up with a dialogue box upon which appears the message that I should "_update the project using SDK tools_". Easy? What does that mean?

What it means is this. I have to:

- Close the project in NetBeans and open a terminal session.
- Navigate to the Android SDK's tools  directory. In there you'll find the `android` utility.
- `./android update project --name MyProjectName --path full_path_to_project`

Job done! Reopen the project in NetBeans and run it. Yippee! By the way, the `--path` parameter is set to the location where the `AndroidManifest.xml` file can be found.

It appears that the Android SDK updated itself thus rendering something that was legal as no longer legal. Nice one - thanks. I'm having enough trouble with Java without having silent updates bollox me as well! ;-)

Cheers.
