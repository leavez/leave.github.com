---
layout: page
title: Design Challenge
permalink: /gsoc/design/
---

shrutirijhwani@gmail.com

Non trivial app challenge TinyWebDB code [here](https://github.com/shrutirij/testdbtiny).

####Part 1: useFront Property

The camera component in App Inventor currently uses an Android Intent in order to access the device camera. This means that the component does not implement the Camera API itself, but simply launches an existing camera app on the device (most likely the default Android camera app), which returns a snapped picture.

The useFront property implemented in ```appinventor-sources/appinventor/components/src/com/google/appinventor/components/runtime/Camera.java``` is used to activate the front camera with the following lines of code

{% highlight java %}
 if (useFront) {
	intent.putExtra("android.intent.extras.CAMERA_FACING", 1);
}
{% endhighlight %}

It adds an extra defined as ```android.intent.extras.CAMERA_FACING``` to the Camera Intent. This extra was never officially documented by Android and was used for testing purposes, according to the Android open source code. 

Device manufacturers were never obligated to implement this Intent extra into their default camera applications. This could be why certain devices do not respond to this property.

As newer Android versions were released, the Camera API was further developed. 

Exploring the [Android Source](https://source.android.com/) allowed an insight into the ```CAMERA_FACING``` Intent extra.


#####Android Gingerbread 2.3.7#####

```CAMERA_FACING``` extra was not present.

The [source code](https://android.googlesource.com/platform/packages/apps/Camera/+/android-2.3.7_r1/src/com/android/camera/Camera.java) that creates the activity for camera functioning does not check for preferred camera.

#####Android Ice Cream Sandwich 4.0.3, 4.0.4#####

 line 82 in [Util.java](https://android.googlesource.com/platform/packages/apps/Camera/+/android-4.0.3_r1/src/com/android/camera/Util.java) contains the definition for the ```CAMERA_FACING``` intent extra (with a 'test only' warning!). 

 {% highlight java %}
 // Private intent extras. Test only.
private static final String EXTRAS_CAMERA_FACING =
"android.intent.extras.CAMERA_FACING";
{% endhighlight %}

 line 492 in [Util.java](https://android.googlesource.com/platform/packages/apps/Camera/+/android-4.0.3_r1/src/com/android/camera/Util.java) has the function ```getCameraFacingIntentExtras``` which checks the ```CAMERA_FACING``` extra and returns the required camera ID. 

 {% highlight java %}
 public static int getCameraFacingIntentExtras(Activity currentActivity) {
	int cameraId = -1;
	int intentCameraId =
	currentActivity.getIntent().getIntExtra(Util.EXTRAS_CAMERA_FACING, -1);
	...
}
{% endhighlight %}


 In [Camera.java](https://android.googlesource.com/platform/packages/apps/Camera/+/android-4.0.3_r1/src/com/android/camera/Camera.java) - 
 ```  onCreate() ``` 
 creates the camera activity, it calls ```  getPreferredCameraId() ``` which in turn calls ``` getCameraFacingIntentExtras() ``` from Util.java (shown above). 

App Inventor's ```useFront``` should work in this Android version, provided the manufacturer has implemented it.

##### Android Jelly Bean 4.1.1 and 4.1.2

A package called LegacyCamera exists with the older Camera API and the Camera package seems to have undergone some development, but the feature in focus ```CAMERA_FACING``` is implemented as in Ice Cream Sandwich.

##### Android Jelly Bean 4.2, 4.2.2, 4.3

[PhotoModule.Java](https://android.googlesource.com/platform/packages/apps/Camera/+/android-4.2_r1/src/com/android/camera/PhotoModule.java) is initialised by [CameraActivity.java](https://android.googlesource.com/platform/packages/apps/Camera/+/android-4.2_r1/src/com/android/camera/CameraActivity.java). ``` init() ``` in PhotoModule calls ``` getCameraFacingIntentExtras() ``` from Util.

App Inventor's ```useFront``` should work in this Android version, as well, provided the manufacturer has implemented it.

##### Android KitKat 4.4, 4.4.2, 4.4.3, 4.4.4 #####

KitKat contains a new package called Camera2. 

[CameraActivity.Java in Camera2](https://android.googlesource.com/platform/packages/apps/Camera2/+/android-4.4.4_r1/src/com/android/camera/CameraActivity.java) calls ```init()``` from ```PhotoModule```, which in turn invokes the function that uses ```CAMERA_FACING```.

However, reading reports of user experience with KitKat devices, it looks like the Camera2 API might have been implemented by most manufacturers without ```CAMERA_FACING```.

##### Android Lollipop 5.0.0, 5.1.0

[PhotoModule.java] (https://android.googlesource.com/platform/packages/apps/Camera2/+/android-5.1.0_r3/src/com/android/camera/PhotoModule.java) **does not call** ``` getCameraFacingIntentExtras() ```. 

The function still exists in the Util, but the Camera2 source **never accesses it**.

Devices running Android Lollipop can not use ```CAMERA_FACING``` and therefore, App Inventor's ```useFront``` will not work.

Android Lollipop is the newest version of the mobile OS. Most new devices will be running Lollipop. As ```CAMERA_FACING``` has been removed from use in the Camera2 API, App Inventor might have to change the implementation of the camera component to achieve the required functionality.

One solution is not depending on the default camera app on devices, but using the Camera/Camera2 API to create an in-built camera in App Inventor applications. After API Level 9, ```CAMERA_FACING_FRONT``` can be used to access the front camera, by apps that directly implement the Camera API.

##### References
[Intents Docs] (http://developer.android.com/reference/android/content/Intent.html), 
[Camera2 Docs] (http://developer.android.com/reference/android/hardware/camera2/package-summary.html),
[Android Source] (https://android.googlesource.com/) for various versions


####Part 2: Automatic Picture Clicking

As mentioned before, the camera component on App Inventor does not implement its own camera application. It uses the default camera application on the device. Because of this fact, the App Inventor component is wholly dependent on how the device camera application is designed. It cannot take advantage of several features in the Android SDK, if they have not been used by the manufacturer of the device.

Taking pictures without user intervention is one such feature. In order to implement it, a new camera component could be introduced in App Inventor - one that implements the Camera API itself, without dependence on other applications. This is necessary because, in order to take a picture automatically, within App Inventor, the picture taking function needs to be available. This is not the case while using the default Android camera app. It is external to any App Inventor application.

This new component - we could call it ```ExtendedCamera``` - will have a method ```AutoPicture``` which, when invoked in a block, automatically takes a picture of whatever view the camera hardware on the device has. The component could also have a property, ```CountDown``` which determines (in seconds) how much time elapses between the method call ```AutoPicture``` and the automatic snapping of the picture. This function can be invoked by any block. 

The component could also have the option to allow the application user to preview the picture and choose to store or delete it. The property ```PreviewEnable``` determines whether the ```ShowPreview``` method should be called after automatically taking the picture. If not, the picture is stored on the device and image path is returned to the calling block.

To an App Inventor user, ```ExtendedCamera``` would be a non-visible component. The user can invoke ```AutoPicture``` at any block and this method will carry out all picture-taking functionality. The app builder can set ```CountDown``` if a time lapse is required before snapping the picture and ```PreviewEnable``` to show the app user a picture preview.

A new activity class ```CameraActivity``` is required to use the Camera API.

On creation, CameraActivity should check for presence of a camera

{% highlight java %}
if (context.getPackageManager().hasSystemFeature(PackageManager.FEATURE_CAMERA)){
    // this device has a camera
    hasCamera = true
} else {
    // no camera on this device
    hasCamera = false
}
{% endhighlight %}

If a camera exists, it should instantiate a camera variable using ```Camera.open()```

Create a dummy ```SurfaceView```.

{% highlight java %}
SurfaceView view = new SurfaceView(this);
cameraObject.setPreviewDisplay(view.getHolder());
cameraObject.startPreview();
{% endhighlight %}

A PictureCallback has to be implemented

{% highlight java %}
 private PictureCallback capturedIt = new PictureCallback() {

      @Override
      public void onPictureTaken(byte[] data, Camera camera) {

      //store the image

      ...
      cameraObject.release();
      }

{% endhighlight %}

Create a picture taking function.

{% highlight java %}
cameraObject.takePicture(null, null, null, PictureCallback);
...
{% endhighlight %}




##### New component 

* ```ExtendedCamera```
 

A new source file at ```appinventor-sources/appinventor/components/src/com/google/appinventor/components/runtime/ExtendedCamera.java``` could be added to implement the new component.

The component uses permission ```android.permission.CAMERA```.

```class ExtendedCamera``` extends ```AndroidNonvisibleComponent```.

This class uses ```CameraActivity```.

It has a helper function ```InitTakePicture()``` which calls the picture taking function from ```CameraActivity```.


##### Properties

* ``` CountDown ``` int, default = 0, ```PropertyTypeConstants.PROPERTY_TYPE_INTEGER``` DesignerProperty

* ```PreviewEnable``` boolean, default = true, ```PropertyTypeConstants.PROPERTY_TYPE_BOOLEAN``` DesignerProperty


Each property to have a getter and setter.

##### Methods

* ```string AutoPicture``` automatically takes a picture after ```countDown``` seconds, returns image path.

* ```bool ShowPreview``` shows a preview of the picture for the user to accept or discard, returns boolean true for accept.

The methods would take advantage of the Camera API from the Android SDK. The API is used to directly access the camera hardware. 

```AutoPicture``` can use the ```CountDownTimer``` class to create a countdown before automatically capturing the image. It calls ```InitTakePicture()```. ```ShowPreview``` should be called if ```PreviewEnable``` is ```true```.

```ShowPreview``` can take the file from the path, preview it and delete it if required. Return ```true``` if accepted, ```false``` if discarded.

```appinventor/build/components/``` files to be modified

* ```simple_components.json``` to be edited with the new component characteristics.
* ```simple_components_build_info.json``` with CAMERA permissions for the new component.
* ```simple_components.txt``` with the component name.

This component can be used with a live feed from the camera, if it is implemented in another component.

This component could be used to manually take pictures as well, thereby getting rid of the Camera Intent pushed to the default Android camera app. 

#####References

[Camera Guide](http://developer.android.com/guide/topics/media/camera.html)




