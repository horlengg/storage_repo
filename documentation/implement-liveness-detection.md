![humnail.png](https://github.com/horlengg/storage_repo/blob/dev/liveness_detection_thumnail.jpeg?raw=true)

<br>

# Liveness Detection

<br>

As digital services continue to replace in-person interactions, 
ensuring that a real, live human is behind every online transaction has never been more critical. 
This is where liveness detection comes in a technology designed to protect identity verification systems from spoofing and fraud.

<br>
<br>

![Liveness Detection Image](/images/liveness_detection_demo.png)

<br>

In this blog, we explore how to implement Liveness Detection in a Flutter app using Google ML Kit’s Face Detection API. 
Liveness detection is a crucial technique in biometric authentication to ensure that the user is a real, live person—not a photo, video, or mask.

If you would like to test it, please download the APK from the following link : 
[Download APK](https://tsfr.io/join/f9u5hy?id=11326582)


<br>

## Contents

1. [Introduction](#introduction)
2. [Google ML Kit Face Detection](#google-ml-kit-face-detection)
3. [Mask Detection](#mask-detection)
3. [Face Anti Spoofing](#face-anti-spoofing)
4. [Implement Liveness Detection](#implement-liveness-detection)
5. [Conclusion](#conclusion)
6. [Full Source Code](#full-source-code)

<br>
<br>

## Introduction

Liveness detection is a biometric security mechanism used to confirm that the person interacting with a system is physically alive and present at the time of verification. In most cases, it’s applied to facial recognition technology, where a camera captures an image or video of a person’s face during onboarding or login processes. But instead of merely comparing the image to a stored template or ID photo, liveness detection takes it one step further—it determines whether that face belongs to a live human being rather than a fraudulent artifact.

<br>
<br>

## Google ML Kit Face Detection

ML Kit Face Detection is a tool for mobile developers that detects faces in images and videos, identifying facial features and contours, and providing information like face orientation and expressions. It's a feature of Google's ML Kit SDK that allows developers to easily integrate advanced face analysis into their apps for use in applications like augmented reality, selfies, and games.

<br>
<br>

## Mask Detection

Mask Detection is a crucial component in the KYC (Know Your Customer) process, ensuring the accurate capture of a user's facial image for identity verification. It detects whether the user is wearing a mask or face covering during the face capture process, enabling the system to prompt users to remove masks if necessary. This ensures that the captured facial data is clear, complete, and suitable for reliable identity validation.

<br>

For more details, please visite this article : [Mask Detection](https://horleng.vercel.app/blogs/implement-mask-detection-for-android)

<br>
<br>

## Face Anti Spoofing

Face Anti Spoofing is a vital security technology integrated into liveness detection systems to verify that the presented face is genuine and live, rather than a printed photo, video, or other counterfeit methods. This feature ensures that the individual attempting authentication is real, enhancing the overall security and reliability of biometric verification processes.

For more details, please visite this article : [Face Anti Spoofing](https://github.com/horlengg/face_anti_spoofing_detector)

<br>
<br>

## Implement Liveness Detection

This comprehensive implementation integrates multiple security features to ensure robust face verification: active liveness detection through blink and open mouth challenges, mask detection to identify masked faces, and face anti-spoofing techniques to prevent presentation attacks.

Key Features:

<br>

- **Time Limitation:** Users have 40 seconds to complete the entire liveness check, ensuring prompt and efficient verification.
- **Device Detection:** Verifies that the device is held vertically (portrait mode) and remains steady during the process, minimizing false detections caused by device movement.
- **Mask Detection:** Detects if the user is wearing a mask; mask usage is not permitted during verification.
- **Face Anti-Spoofing:** Implements advanced techniques to identify print photos, video replays, or other presentation attacks, ensuring the face is live and genuine.
- **Challenge Verification:** Requires the user to perform specific actions—blink and open mouth—to actively confirm liveness.
- **Capture for KYC:** Upon successful verification, captures and stores the user's face for Know Your Customer (KYC) purposes.

<br>

Here is some implementation code...

<br>
<br>

<i>face_detection_helper.dart</i>

```dart

class Challenge {
  final String instruction;
  final String instructionImageName;
  final bool Function(Face face) verify;
  Challenge(this.instruction,this.instructionImageName, this.verify);
}

class FaceDetectionHelper {

  static List<Challenge> getChallengeList(){
    final challengeList = [
      Challenge(
        "Please blink slowly",
        "blink.png",
        BlinkDetector.instance.detectBlink
      ),
      Challenge(
        "Please open your mouth",
        "open_mouth.png",
        _isMouthOpen,
      ),
    ];
    challengeList.shuffle();
    return challengeList;
  }

  static bool isFaceFullyVisibleInCircle({
    required Rect boundingBox,
    required Size cameraSize,
    required Size widgetSize,
    required double cameraRatio
  }) {

    final scaleX = widgetSize.width / cameraSize.width;
    final scaleY = widgetSize.height / cameraSize.height;

    final scaledBox = Rect.fromLTWH(
      boundingBox.left * scaleX,
      boundingBox.top * scaleY,
      boundingBox.width * scaleX,
      boundingBox.height * scaleY,
    );

    final fullyVisible =
        scaledBox.left >= 0 &&
        scaledBox.top >= 0 &&
        (scaledBox.right * cameraRatio) <= widgetSize.width &&
        (scaledBox.bottom / cameraRatio) <= widgetSize.height;
    return fullyVisible;
  }


  Future<double> getApplicationBrightness() async{
    try {
      return await ScreenBrightness.instance.application;
    } catch (e) {
      throw 'Failed to get application brightness';
    }
  }

  Future<void> setApplicationBrightness(double brightness) async {
    try {
      await ScreenBrightness.instance.setApplicationScreenBrightness(brightness);
    } catch (e) {
      log(e.toString());
      throw 'Failed to set application brightness';
    }
  }

  static CameraStreamPayload? handleCaptureFaceForKYC(Face face,CameraStreamPayload payload) {
    // Check eyes are open
    if (face.leftEyeOpenProbability == null || 
        face.rightEyeOpenProbability == null ||
        face.leftEyeOpenProbability! < 0.6 ||
        face.rightEyeOpenProbability! < 0.6) {
      log("Eyes not fully open");
      return null;
    }

    // Check head tilt (Z-axis)
    if (face.headEulerAngleZ == null || face.headEulerAngleZ!.abs() > 10) {
      log("Head tilted too much: ${face.headEulerAngleZ?.abs()}");
      return null;
    }

    // Check horizontal rotation (Y-axis)
    if (face.headEulerAngleY == null || face.headEulerAngleY!.abs() > 10) {
      log("Head rotated horizontally too much: ${face.headEulerAngleY?.abs()}");
      return null;
    }

    // Optional: Check vertical rotation (X-axis)
    if (face.headEulerAngleX != null && face.headEulerAngleX!.abs() > 10) {
      log("Head tilted up/down too much: ${face.headEulerAngleX?.abs()}");
      return null;
    }

    if(_isMouthOpen(face,threshold: 10)) return null; 
    
    return payload;

  }

  static bool _isMouthOpen(Face face,{
    int threshold = 25
  }){
    final upperLip = face.contours[FaceContourType.upperLipBottom]?.points;
    final lowerLip = face.contours[FaceContourType.lowerLipTop]?.points;
    if (upperLip != null && lowerLip != null && upperLip.isNotEmpty && lowerLip.isNotEmpty) {
      final topCenter = upperLip[upperLip.length ~/ 2];
      final bottomCenter = lowerLip[lowerLip.length ~/ 2];
      final gap = (bottomCenter.y - topCenter.y).abs();
      log("Mouth open :::: $gap");
      return gap > threshold;
    }
    return false;
  }

}
```

<br>
<br>

<i>device_motion_detector.dart</i>

```dart

class DeviceMotionDetector {

  static final DeviceMotionDetector _instance = DeviceMotionDetector._internal();
  
  DeviceMotionDetector._internal();
  
  static DeviceMotionDetector get instance => _instance;

  double _lastX = 0, _lastY = 0, _lastZ = 0;
  int _lastTimestamp = 0;
  bool _isFirstReading = true;
  StreamSubscription<AccelerometerEvent>? _accelerometerSubscription;
  bool _isDevicePositionVertical = false;
  bool _isDeviceMoving = false;

  bool get isDevicePositionVertical => _isDevicePositionVertical;
  bool get isDeviceMoving => _isDeviceMoving;

  void _detectDeviceVertical(AccelerometerEvent event,Function(bool value)? deviceVerticalChangeCallback) {
    
    double x = event.x;
    double y = event.y;
    double z = event.z;
    
    // Calculate total tilt from vertical using pitch and roll
    double pitch = math.atan2(x, math.sqrt(y * y + z * z)) * (180 / math.pi);
    double roll = math.atan2(y, math.sqrt(x * x + z * z)) * (180 / math.pi);
    
    // Total tilt angle
    double tiltAngle = math.sqrt(pitch * pitch + roll * roll);
    
    if(tiltAngle > 60){
      if(!_isDevicePositionVertical){
        _isDevicePositionVertical = true;
        deviceVerticalChangeCallback?.call(_isDevicePositionVertical);
      }
    }else {
      if(isDevicePositionVertical){
        _isDevicePositionVertical = false;
        deviceVerticalChangeCallback?.call(_isDevicePositionVertical);
      }
    }
    

  }

  void _detectDeviceMoving(
    AccelerometerEvent event,
    Function(bool value)? deviceMovingChangeCallback
  ) {
    if (_isFirstReading) {
      _lastX = event.x;
      _lastY = event.y;
      _lastZ = event.z;
      _lastTimestamp = DateTime.now().millisecondsSinceEpoch;
      _isFirstReading = false;
      return;
    }
    
    // Calculate time delta
    int currentTime = DateTime.now().millisecondsSinceEpoch;
    double timeDelta = (currentTime - _lastTimestamp) / 1000.0; // in seconds
    
    if (timeDelta == 0 || timeDelta > 0.5) {
      // Skip if time is invalid or too long (app was paused)
      _lastX = event.x;
      _lastY = event.y;
      _lastZ = event.z;
      _lastTimestamp = currentTime;
      return;
    }
    
    // Calculate change in acceleration (delta)
    double deltaX = (event.x - _lastX).abs();
    double deltaY = (event.y - _lastY).abs();
    double deltaZ = (event.z - _lastZ).abs();
    
    // Calculate velocity (rate of change per second)
    double velocityX = deltaX / timeDelta;
    double velocityY = deltaY / timeDelta;
    double velocityZ = deltaZ / timeDelta;
    
    // Calculate total velocity magnitude
    double velocity = math.sqrt(
      velocityX * velocityX + 
      velocityY * velocityY + 
      velocityZ * velocityZ
    );
    
    const double blurThreshold = 5.0;
    
    bool isFastMovement = velocity > blurThreshold;
    
    if (isFastMovement != _isDeviceMoving) {
      _isDeviceMoving = isFastMovement;
      deviceMovingChangeCallback?.call(_isDeviceMoving);
    }
    
    // Update last values
    _lastX = event.x;
    _lastY = event.y;
    _lastZ = event.z;
    _lastTimestamp = currentTime;
  }

  void setup({
    Function(bool value)? deviceMovingChangeCallback,
    Function(bool value)? deviceVerticalChangeCallback,
  }) {
    // Listen to accelerometer events
    _accelerometerSubscription = accelerometerEventStream().listen((AccelerometerEvent event) {
        _detectDeviceMoving(event,deviceMovingChangeCallback);
        _detectDeviceVertical(event,deviceVerticalChangeCallback);
      },
    );
  }

  void destroy(){
    _accelerometerSubscription?.cancel();
    _accelerometerSubscription = null;
    _isDeviceMoving = false;
    _isDevicePositionVertical = false;
  }

}

```

<br>
<br>

<i>Initialize model...</i>

```dart
@override
  void initState() {
    super.initState();
    _challengeList = FaceDetectionHelper.getChallengeList();
    _initDetection();
     SystemChrome.setPreferredOrientations([
      DeviceOrientation.portraitUp,
    ]);

    // _startAccelerometerListener();
    DeviceMotionDetector.instance.setup(
      deviceMovingChangeCallback: (value) {
        setState(() {
          _isDeviceMoving = value;
        });
      },
      deviceVerticalChangeCallback: (value) {
        setState(() {
          _isCorrectDevicePosition = value;
        });
      },
    );

    setTimeoutTracking();

  }
```

<br>
<br>

<i>Clean up memory...</i>

```dart
@override
  void dispose() {
    _isWidgetDestroyed = true;
    _faceDetector.close();
    MaskDetector.destroy();
    FaceAntiSpoofingDetector.destroy();
    super.dispose();
    DeviceMotionDetector.instance.destroy();
    BlinkDetector.instance.reset();
    timer?.cancel();
  }
```

<br>
<br>

<i>Main function check Liveness...</i>

```dart

Future<void> _detectFrameCameraStream(CameraStreamPayload payload) async {

   if(_isDetectionOnProcessing || _isWidgetDestroyed) return;

   _isDetectionOnProcessing = true;
   
   final request = _challengeList[_currentStep];
   log(request.instruction);

   try {

   final startDate = DateTime.now();

   final faces = await _faceDetector.processImage(payload.inputImage);
   // 
   _faceValidation(faces);

   final bx = faces[0].boundingBox;

   final faceContour = Rect.fromLTRB(bx.left, bx.top, bx.right,bx.bottom);
   final maskResult = await MaskDetector.detect(
      payload.yuvBytes, 
      imageWidth: payload.imageWidth.toDouble(), 
      imageHeight: payload.imageHeight.toDouble(), 
      faceCountour: faceContour,
      rotation: payload.rotation
   );

   if(maskResult.hasMask){
      throw LivenessCheckException("Please turn off your mask!.");
   }

   final confidenceScore = await FaceAntiSpoofingDetector.detect(
      yuvBytes: payload.yuvBytes, 
      previewWidth: payload.imageWidth, 
      previewHeight: payload.imageHeight, 
      orientation: 7, 
      faceContour: faceContour
   );
   
   if(confidenceScore == null || confidenceScore < .95){
      throw LivenessCheckException("A real person is required for liveness verification.");
   }
   
   log("============================");
   log("duration : ${DateTime.now().difference(startDate).inMilliseconds} ms");
   log("============================");

   // capture face
   if(_cameraFrameCaptured == null){
      _userValidationValidAt ??= DateTime.now();
      final timeSinceValid = DateTime.now().difference(_userValidationValidAt!).inMilliseconds;
      log("timeSinceValid : $timeSinceValid");
      if(timeSinceValid >= 1200){
         _cameraFrameCaptured = FaceDetectionHelper.handleCaptureFaceForKYC(faces.first, payload);
         log("Face captured after ${timeSinceValid}ms of continuous validation");
      }
   }
   
   _faceVerification(
      face : faces.first,
      request : request
   );

   _warningMsg = null;
   
   } on LivenessCheckException catch (e){
   _warningMsg = e.toString();
   log(e.what());
   BlinkDetector.instance.reset();
   _userValidationValidAt = null;
   } catch (e) {
   log("Error : $e");
   _warningMsg = null;
   } finally {
   _isDetectionOnProcessing = false;
   setState(() {});
   }
}

```

<br>
<br>

## Conclusion

Implementing Liveness Detection using Flutter and Google ML Kit is not only possible but also efficient for many real-world mobile applications, especially those that require secure and user-friendly identity verification. By leveraging facial landmarks and creating custom logic for verifying actions like blinking or head movement, you can build a reliable layer of biometric security without relying on costly third-party services.

This tutorial demonstrated how to detect live human presence using the built-in face detection capabilities of ML Kit and Flutter’s real-time camera integration. With a bit of customization, you can expand this foundation to support more advanced or hybrid liveness strategies.

<br><br>

## 📦 Full Source Code

You can find the complete source code and example implementation on GitHub:  [https://github.com/horlengg/liveness_detection](https://github.com/horlengg/liveness_detection)

<br><br>

<i>Thank you guys for reading this blog!</i>

<br><br><br><br><br>