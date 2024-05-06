---
layout: post
title:  "Befriending Unity and ARKit"
date:   2017-06-06 00:06:31
description: 'How to combine leading AR technologies in one iOS app'
---

## Disclaimer
This document was written in 2017 and conents may have became outdated since.

## Introduction
Nowadays people start to use AR and VR in various types of everyday activities and there is no doubt that augmented reality is the future of human interaction with the virtual world. But from a developer side, there still are important choices that can make development easier and the final product better or vice versa. Our team also faced such choices and we want to share our experience.

## Alternatives overview
Apple’s ARKit was released plenty of time ago, and it is still a very simple and powerful tool to build AR experience on iOS. Despite having plenty of alternatives, ARKit is free and suitable for basic needs. But ARKit is only a tool for providing user position and plane detection, so we still need a framework to deal with 3D objects, animations, and interaction with them. Apple offers SceneKit for this, but Xcode has very limited functionality for creating and editing 3D scenes, so we decided to use Unity instead. As a complete 3D game engine it provides all needed tools for any person, even not familiar with programming. Unity nicely suits for apps, where emphasis is placed on beautiful 3D models, scenes and effects, so we can highlight such pros of using Unity:
- Has much more tools for editing 3D models
- Easier work with animations
- More helpful features of using 3D scenes  

But if your app doesn’t need advanced Unity features, then you can use SceneKit to avoid some problems and inconveniences. Comparing to native Apple frameworks, Unity has following cons:
- Heavyweight app
- Long and complex build process
- Harder to manage UI structure
- Hard interaction of native and unity code
- Dependence on Unity and plugins developers  

To understand some of the reasons above better, we need to dive into development process. At the start of our project our team decided to use Unity to make life easier for our designers and 3D content makers. So now we will show you main steps of creating simple iOS AR app with Unity and try make things clearer about good and bad aspects of such applications development. 

## Using AR in Unity
ARKit in Unity is provided by the Unity iOS ARKit plugin (bitbucket.org/Unity-Technologies/unity-arkit-plugin). It can be downloaded for free from unity asset store, including examples and readme guide. To add it to your scene you just need to add several scripts to camera and set camera settings as below.

![img1](https://eknm-hub-public.s3.eu-central-1.amazonaws.com/arkit_unity/image1.jpg)

## Adding native code
When you first build a Unity project for iOS it creates an Xcode project and then this project is built. Every next time unity can append the Xcode project, instead of replacing, but some files need to be overwritten anyway. According to documentation, there are two ways to add custom sources to project:
- .Add sources to generated Xcode project and store them in the Classes folder, which is not overwritten during following Unity builds
- Create native plugin for iOS, that will be added to Xcode project every new build  

.Obviously the second way is much better, because the first one has serious problems with source control: generated sources and XCode files must be ignored, and all new custom sources and resources must be added to the project manually. Also a build can not be easily reproduced from scratch.

Plugin files must be located in `Assets/Plugins/iOS` folder, but subfolders are not supported (so hard to support it!) and only some file extensions are supported, for example .jpg image won’t be copied and added to project, unlike .png image. This means, that some files, such as non-standard resources, need to be copied in another way. 
Also some things must be done every build:
- Project settings, build stages, included files must be set
- Plist must be appended with custom values
- Non-standard resources, referenced above must be copied and added to project
- Third-party libraries and frameworks must be added to project  

This can be done with Xcode Manipulation API (bitbucket.org/Unity-Technologies/xcodeapi) by Unity iOS team. The code can be written in `XcodePostprocess.cs` inside `OnPostprocessBuild (buildTarget, path)` function.
For example to add build property to project:

{% highlight csharp %}
string projPath = Path.Combine(path,Path.Combine("Unity-iPhone.xcodeproj", "project.pbxproj"));
PBXProject project = new PBXProject();
project.ReadFromString (File.ReadAllText (projectPath));
string target = project.TargetGuidByName ("Unity-iPhone");
proj.AddBuildProperty (target, "HEADER_SEARCH_PATHS", "$(inherited)");
File.WriteAllText (projectPath, project.WriteToString ());
{% endhighlight %}

To add a file to project:
{% highlight csharp %}
File.Copy (Path.Combine("FileLocation”,”resource.json"), Path.Combine("LocationInProject”, “resource.json"), true);
string fileReference = proj.AddFile (Path.Combine (path, Path.Combine("LocationInProject", "resource.json")), Path.Combine("LocationInProject", "resource.json"));
proj.AddFileToBuild(target, fileReference);
{% endhighlight %}

To add something to plist:
{% highlight csharp %}
string plistPath = Path.Combine(path, "Info.plist");
string newPlistPath = Path.Combine(path, "newInfo.plist");
if (!File.Exists(plistPath) && File.Exists(newPlistPath)) {
    File.Move(newPlistPath, plistPath);  
}
PlistDocument plist = new PlistDocument ();
plist.ReadFromString (File.ReadAllText (plistPath));
PlistElementDict rootDict = plist.root;
rootDict.SetString("key","value");
File.WriteAllText(newPlistPath, plist.WriteToString());
File.Delete (plistPath);
File.Move(newPlistPath, plistPath); 
{% endhighlight %}

Unfortunately, this API has poor documentation and almost no other information resources, so these lines of code can be useful to start with and then do other things similarly.  
The most popular way to add third-party libraries to a project is to use cocoapods. All we need to do is to copy the podfile to the project (using code above) and to run the `pod install` command in that folder after the XCode project is generated. The good news is there’s no need to run it every time you generate an Xcode project, but you can always write a script to do it faster. Now, when we have all required components of the project, it remains only to use them properly.

### Controllers interaction
iOS native controllers are very useful almost in any case (except when you create games). Unity doesn’t provide native UI elements from UIKit, although almost everything can be done with various Unity plugins, but it’s like building a motorcycle from parts of car and bicycle. So the best way is combining unity controllers with native controllers.

![img2](https://eknm-hub-public.s3.eu-central-1.amazonaws.com/arkit_unity/image2.jpg)

When your app launches, the window’s root view controller is set to the *UnityDefaultViewController* instance. If app’s main functionality will be presented by native controllers, then the best decision will be to make your app’s main controller root, save Unity controller and just push it when needed:

{% highlight objc %}
UIWindow * window = [[[UIApplication sharedApplication] delegate] window];
CustomRootViewController * customRootVC = [[CustomRootViewController alloc] init];
UIViewController * unityViewController = [window rootViewController];
[window setRootViewController:customRootVC];
{% endhighlight %}

Definitely we will need to call some Unity scripts from the native code, for this purpose we can use UnitySendMessage function:
{% highlight csharp %}
UnitySendMessage("GameObjectName", "MethodName", "Message to send")
{% endhighlight %}

This function is trying to call MethodName for all scripts, added to the object named GameObjectName on the current scene, so be sure that only script you need has MethodName function. Unfortunately we can pass only string argument to Unity script, so if you need to send some complex object, you can convert it to json format and send a string with it:

{% highlight objc %}
NSMutableDictionary * dict = [[NSMutableDictionary alloc] init];
[dict setObject:@”value” forKey:@"key"];
NSData *jsonData = [NSJSONSerialization dataWithJSONObject:dict
                               options:NSJSONWritingPrettyPrinted                                                         error:nil];
NSString * string = [[NSString alloc] initWithData:jsonData encoding:NSUTF8StringEncoding];
const char * cstring = [string UTF8String];
UnitySendMessage("YourUnityObject", "YourUnityFunction", cstring);
{% endhighlight %}

and convert it back in Unity script using any way you like.  

Sending data from Unity code to native is easier, but has its own problems. You need to implement a C function in native code, and declare it in C# code, using `[DllImport ("__Internal")]` before declaration. If a C function is implemented in C++ or Objective-C++ file you need to wrap it in `extern "C" {}`. This is not said in documentation, but even in Objective-C functions with multiple arguments suffer from name mangling issues, so it is better to wrap all functions in extern "C" {}. For example if we need to send current camera position to native code:

{% highlight csharp %}
=== YourScript.cs ===
[DllImport("__Internal")]
public static extern void ARCameraCoordinateUpdated(float x, float y, float z);
=== YourNativeSource.h === 
extern "C" {
    void ARCameraCoordinateUpdated(float x, float y, float z);
}
{% endhighlight %}

### Conclusion
We walked through the main steps of creating an AR iOS project with Unity and any functionality, provided by Apple and Xcode. Also they uncover some details, specific to Unity and important to understand before choosing to use, or not to use it. Such symbiosis can add much needed functionality to your project at the price of increased technical complexity.
I hope my experience will help you to make right decisions and create good applications.
