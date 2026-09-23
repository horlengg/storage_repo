
![humnail.png](https://github.com/horlengg/storage_repo/blob/dev/mask-detection-demo.jpeg?raw=true)

<br>

# Mask Detection

<br>

Hello guys!

*Welcome to my blog. In this article, I want to share my exploration of a mask detection project developed with Flutter* 
*which can be helpful for KYC processes involving user face verification, such as Liveness Detection and Face Recognition.*

<br>
<br>

![demo.png](/images/mask_demo.jpg)


<br>

If you would like to test it, please download the APK from the following link : 
[Download APK](https://tsfr.io/join/f9u5hy?id=11347182)

---

<br>
<br>

## Technologies Used

- **Flutter**: Cross-platform UI framework for building the mobile application.
- **TensorFlow Lite**: Lightweight machine learning model framework for excecute model on device.
- **Mask Detection Model**: Existing ML Model public by [https://github.com/chandrikadeb7/Face-Mask-Detection](https://github.com/chandrikadeb7/Face-Mask-Detection)
- **Google MLKit Face Detection**: Official Google ML for detect face from rgb image
- **Camera Plugin**: Flutter plugin (`camera`) for accessing device camera and capturing frames.


<br>
<br>

## Integrating the Model

For this project, I utilized a model available from [this GitHub repository](). The model is designed for face mask detection and is optimized for Android devices, making it suitable for integration into Kotlin via TensorFlow Lite.

Here is my native code for integration with Mask Detection Model :


***MaskDetector.kt***
```kotlin

package com.lengdev.maskdetector

import android.content.Context
import android.graphics.*
import android.media.Image
import android.os.Build
import org.tensorflow.lite.Interpreter
import org.tensorflow.lite.support.image.ImageProcessor
import org.tensorflow.lite.support.image.TensorImage
import org.tensorflow.lite.support.image.ops.ResizeOp
import org.tensorflow.lite.support.image.ops.ResizeWithCropOrPadOp
import org.tensorflow.lite.support.label.TensorLabel
import org.tensorflow.lite.support.tensorbuffer.TensorBuffer
import org.tensorflow.lite.support.common.ops.NormalizeOp
import org.tensorflow.lite.support.common.FileUtil
import java.io.ByteArrayOutputStream
import java.io.File
import java.nio.ByteBuffer
import kotlin.math.min
import android.util.Log

class MaskDetector(private val context: Context) {

    companion object {
        private const val TAG = "MaskDetector"
    }

    private var model: Interpreter? = null
    private var imageProcessor: ImageProcessor? = null
    private val labels = listOf("WithMask", "WithoutMask")

    /**
     * Initialize the model and image processor
     * Call this once when starting the detector
     */
    fun initialize() : Boolean {
        try {
            val modelFile = FileUtil.loadMappedFile(context, "mask_detector_v3.tflite")
            model = Interpreter(modelFile, Interpreter.Options())

            // Get input shape to create image processor
            val inputShape = model?.getInputTensor(0)?.shape()
            if (inputShape != null && inputShape.size >= 3) {
                imageProcessor = ImageProcessor.Builder()
                    .add(ResizeWithCropOrPadOp(inputShape[1], inputShape[2]))
                    .add(ResizeOp(inputShape[1], inputShape[2], ResizeOp.ResizeMethod.NEAREST_NEIGHBOR))
                    .add(NormalizeOp(127.5f, 127.5f))
                    .build()
            }
            
            Log.d(TAG, "Model initialized successfully")
            return true;
        } catch (e: Exception) {
            Log.e(TAG, "initialize() error: ${e.message}")
            return false;
        }
    }

    /**
     * Clean up resources
     * Call this when done with the detector
     */
    fun destroy():Boolean {
        try {
            model?.close()
            model = null
            imageProcessor = null
            Log.d(TAG, "Model destroyed successfully")
            return true;
        } catch (e: Exception) {
            Log.e(TAG, "destroy() error: ${e.message}")
            return false;
        }
    }

    /**
     * Detect mask from YUV image data (from Flutter camera)
     */
    fun detectMask(
        yuvBytes: ByteArray,
        width: Int,
        height: Int,
        rotation: Int,
        rect: FaceContour
    ): MaskDetectionResult? {
        try {

            val startAt = System.currentTimeMillis()
            // Check if model is initialized
            if (model == null) {
                Log.e(TAG, "Model not initialized. Call initialize() first.")
                return null
            }
            
            // Convert YUV to Bitmap
            val bitmap = yuv420ToBitmap(yuvBytes, width, height)
            
            // Rotate bitmap if needed
            val rotatedBitmap = if (rotation != 0) {
                rotateBitmap(bitmap, rotation.toFloat())
            } else {
                bitmap
            }

            // Get dimensions
            val bmpWidth = rotatedBitmap.width
            val bmpHeight = rotatedBitmap.height

            // Clamp coordinates to be within bitmap bounds
            val x = rect.left.coerceIn(0, bmpWidth - 1)
            val y = rect.top.coerceIn(0, bmpHeight - 1)
            val widthCrop = rect.width.coerceAtMost(bmpWidth - x)
            val heightCrop = rect.height.coerceAtMost(bmpHeight - y)

            val croppedFace = Bitmap.createBitmap(rotatedBitmap, x, y, widthCrop, heightCrop)


            // Predict mask
            val label = predict(croppedFace)

            val withMask = label["WithMask"] ?: 0f
            val withoutMask = label["WithoutMask"] ?: 0f

            val hasMask = withMask > withoutMask

            return MaskDetectionResult(
                hasMask = hasMask,
                withMaskScore = withMask,
                withoutMaskScore = withoutMask,
                durationInMilliseconds = (System.currentTimeMillis() - startAt).toFloat()
            )
            
        } catch (e: Exception) {
            Log.e(TAG, "detectMask() error: ${e.message}")
            return null
        }
    }

    /**
     * Convert YUV420 to Bitmap
     */
    private fun yuv420ToBitmap(yuvBytes: ByteArray, width: Int, height: Int): Bitmap {
        val yuvImage = YuvImage(yuvBytes, ImageFormat.NV21, width, height, null)
        val out = ByteArrayOutputStream()
        yuvImage.compressToJpeg(Rect(0, 0, width, height), 100, out)
        val imageBytes = out.toByteArray()
        return BitmapFactory.decodeByteArray(imageBytes, 0, imageBytes.size)
    }

    /**
     * Rotate bitmap
     */
    private fun rotateBitmap(bitmap: Bitmap, degrees: Float): Bitmap {
        val matrix = Matrix()
        matrix.postRotate(degrees)
        return Bitmap.createBitmap(bitmap, 0, 0, bitmap.width, bitmap.height, matrix, true)
    }

    /**
     * Run prediction on the input bitmap
     */
    private fun predict(input: Bitmap): MutableMap<String, Float> {
        val currentModel = model ?: throw IllegalStateException("Model not initialized")
        val currentProcessor = imageProcessor ?: throw IllegalStateException("Image processor not initialized")

        val imageDataType = currentModel.getInputTensor(0).dataType()
        val outputDataType = currentModel.getOutputTensor(0).dataType()
        val outputShape = currentModel.getOutputTensor(0).shape()

        var inputImageBuffer = TensorImage(imageDataType)
        val outputBuffer = TensorBuffer.createFixedSize(outputShape, outputDataType)

        inputImageBuffer.load(input)
        inputImageBuffer = currentProcessor.process(inputImageBuffer)

        currentModel.run(inputImageBuffer.buffer, outputBuffer.buffer.rewind())

        val labelOutput = TensorLabel(labels, outputBuffer)
        
        return labelOutput.mapWithFloatValue
    }
}

data class FaceContour (
    val left : Int,
    val top : Int,
    val width : Int,
    val height : Int
)

```

<br>
<br>

***MaskClassifier.swift***

```swift
//
//  MaskClassifier.swift
//  MaskDetectionApp
//
//  Created by Houleng Ly on 18/5/26.
//

import Foundation
import TensorFlowLite
import UIKit
import CoreImage

class MaskClassifier {
    private var interpreter: Interpreter?

    let inputSize = 224

    init() {
        loadModel()
    }

    private func loadModel() {
        do {
            let bundle = Bundle(for: MaskDetectorPlugin.self)
            let resourceBundle = bundle.url(forResource: "mask_detector", withExtension: "bundle")
                .flatMap { Bundle(url: $0) } ?? bundle

            guard let modelPath = resourceBundle.path(
                forResource: "mask_detector_v3",
                ofType: "tflite"
            ) else { 
                print("mask_detector_v2.tflite not found")
                return
            }
            var options = Interpreter.Options()
            options.threadCount = 2
            let interp = try Interpreter(modelPath: modelPath, options: options)
            try interp.allocateTensors()
            let inputTensor = try interp.input(at: 0)
            interpreter = interp
        } catch let error as InterpreterError {
            print("❌ Interpreter error: \(error.localizedDescription)")
        } catch {
            print("❌ Unexpected error loading model: \(error)")
        }
    }

    func predict(image: UIImage) -> MaskResponse? {
        guard let interpreter = interpreter else {
            print("Interpreter not loaded")
            return nil
        }
        guard let pixelBuffer = preprocessImage(image) else { return nil }

        do {
            

            try interpreter.copy(pixelBuffer, toInputAt: 0)
            try interpreter.invoke()

            let outputTensor = try interpreter.output(at: 0)
            let results: [Float] = outputTensor.data.withUnsafeBytes {
                Array($0.bindMemory(to: Float.self))
            }
            print("results : \(results)")
            return MaskResponse(
                mask: results[0],
                withoutMask: results[1],
                durationInMilliseconds: 0.0
            )
        } catch {
            print("Inference error: \(error)")
            return nil
        }
    }

    private func preprocessImage(_ image: UIImage) -> Data? {
        guard let cgImage = image.cgImage else {
            print("No cgImage")
            return nil
        }
        
        let width = inputSize
        let height = inputSize
        let bytesPerRow = width * 4

        var rawBytes = [UInt8](repeating: 0, count: bytesPerRow * height)
        let colorSpace = CGColorSpaceCreateDeviceRGB()

        guard let context = CGContext(
            data: &rawBytes,
            width: width,
            height: height,
            bitsPerComponent: 8,
            bytesPerRow: bytesPerRow,
            space: colorSpace,
            bitmapInfo: CGImageAlphaInfo.noneSkipLast.rawValue
        ) else {
            print("CGContext failed")
            return nil
        }

        context.draw(cgImage, in: CGRect(x: 0, y: 0, width: width, height: height))

        var floatData = [Float]()
        floatData.reserveCapacity(width * height * 3)

        for i in 0 ..< width * height {
            let base = i * 4
            floatData.append((Float(rawBytes[base])     - 127.5) / 127.5)  // R
            floatData.append((Float(rawBytes[base + 1]) - 127.5) / 127.5)  // G
            floatData.append((Float(rawBytes[base + 2]) - 127.5) / 127.5)  // B
        }

        return Data(bytes: floatData, count: floatData.count * MemoryLayout<Float>.size)
    }
}

struct MaskResponse {
    var mask: Float
    var withoutMask: Float
    var durationInMilliseconds: TimeInterval
}

```

<br>
<br>

## Implementing Real Time Face Mask Detection

This project is developed using Flutter. To access native platform-specific features, such as predict mask, we utilize MethodChannel to communicate between Flutter and native code.


***MaskDetectorPlugin.kt***

```kotlin

/** MaskDetectorPlugin */
class MaskDetectorPlugin: FlutterPlugin, MethodCallHandler {
  /// The MethodChannel that will the communication between Flutter and native Android
  ///
  /// This local reference serves to register the plugin with the Flutter Engine and unregister it
  /// when the Flutter Engine is detached from the Activity
  private lateinit var channel : MethodChannel
  private var maskDetector : MaskDetector? = null
  private var TAG = "MaskDetector"
  private lateinit var context : Context

  override fun onAttachedToEngine(flutterPluginBinding: FlutterPlugin.FlutterPluginBinding) {
    context = flutterPluginBinding.applicationContext
    channel = MethodChannel(flutterPluginBinding.binaryMessenger, "com.lengdev.maskdetector")
    channel.setMethodCallHandler(this)
  }

  override fun onMethodCall(call: MethodCall, result: Result) {
        when (call.method) {
            "initialize" -> handleInitialize(call, result)
            "detectMask" -> handleDetectMask(call, result)
            "destroy" -> handleDestroy(call, result)
            else -> result.notImplemented()
        }
    }

    private fun handleDetectMask(call: MethodCall, result: Result) {
        
        val yuvBytes = call.argument<ByteArray>("yuvBytes") 
            ?: throw IllegalArgumentException("Missing yuvBytes")
        val width = call.argument<Int>("imageWidth") 
            ?: throw IllegalArgumentException("Missing width")
        val height = call.argument<Int>("imageHeight") 
            ?: throw IllegalArgumentException("Missing height")
        val rotation = call.argument<Int>("rotation") 
            ?: throw IllegalArgumentException("Missing rotation")
        val faceContour = call.argument<Map<String, Int>>("faceContour") ?: throw IllegalArgumentException("Missing faceBoundingBox")

        val left = faceContour["left"] ?: 0
        val top = faceContour["top"] ?: 0
        val right = faceContour["right"] ?: 0
        val bottom = faceContour["bottom"] ?: 0

        val faceBoundingBox = FaceContour(
            left = left,
            top = top,
            width = right - left,
            height = bottom - top
        )

        // Perform heavy mask detection
        val response = maskDetector?.detectMask(
            yuvBytes, 
            width, 
            height, 
            rotation,
            faceBoundingBox
        )

        if(response == null){
            result.error("PREDICTION_ERROR", "Detect mask failed", null)
        }else {
            result.success(mapOf(
                "hasMask" to response.hasMask,
                "withMaskScore" to String.format("%.4f", response.withMaskScore).toDouble(),
                "withoutMaskScore" to String.format("%.4f", response.withoutMaskScore).toDouble(),
            ))

        }
        
    }
    fun handleInitialize(call: MethodCall, result: Result){

        if(maskDetector != null){
            maskDetector?.destroy()
            maskDetector = null
        }

        maskDetector = MaskDetector(context)
        val status = maskDetector?.initialize()

        if(status == true){
            result.success(mapOf(
                "status" to true,
                "message" to "Initialize mask detector success!."
            ))
        }else {
            result.error("INITIALIZE_FAIL","Failed to initialize mask detector!.",null)
        }


    }

    fun handleDestroy(call: MethodCall, result: Result){
        if(maskDetector != null){
            val status = maskDetector?.destroy()
            if(status == true){
                result.success(mapOf(
                    "status" to true,
                    "message" to "Mask detector was destroyed!."
                ))
            }else {
                result.error("DESTROY_FAILED","Failed to destroy mask detector!.",null)
            }
        }else {
            result.error("DESTROY_FAILED","Failed to destroy mask detector!.",null)
        }
    }

    override fun onDetachedFromEngine(binding: FlutterPlugin.FlutterPluginBinding) {
        channel.setMethodCallHandler(null)
    }
}


```
<br>
<br>


***MaskDetectorPlugin.swift***

```swift
//
//  MaskDetectorPlugin.swift
//  Runner
//
//  Created by Houleng Ly on 23/5/26.
//
import Flutter
import UIKit

public class MaskDetectorPlugin: NSObject, FlutterPlugin {

  private var classifier: MaskClassifier?

  public static func register(with registrar: FlutterPluginRegistrar) {
    let channel = FlutterMethodChannel(
      name: "com.lengdev.maskdetector",
      binaryMessenger: registrar.messenger()
    )
    let instance = MaskDetectorPlugin()
    registrar.addMethodCallDelegate(instance, channel: channel)
  }

  public func handle(_ call: FlutterMethodCall, result: @escaping FlutterResult) {
    switch call.method {
    case "initialize":  handleInitialize(result: result)
    case "detectMask":  handleDetectMask(call: call, result: result)
    case "destroy":     handleDestroy(result: result)
    default:            result(FlutterMethodNotImplemented)
    }
  }

  private func handleInitialize(result: @escaping FlutterResult) {
    DispatchQueue.global(qos: .userInitiated).async {
      self.classifier = MaskClassifier()
      let success = self.classifier != nil
      DispatchQueue.main.async {
        if success {
          result([
            "status" : true,
            "message" : "Initialize mask detector success!."
          ])
        } else {
          result(FlutterError(
            code: "INIT_FAILED",
            message: "Failed to load mask_detector.tflite",
            details: nil
          ))
        }
      }
    }
  }


  private func handleDestroy(result: @escaping FlutterResult) {
    classifier = nil
    result([
      "status" : true,
      "message" : "Mask detector was destroyed!."
    ])
    
  }
    
  private func handleDetectMask(call: FlutterMethodCall, result: @escaping FlutterResult) {
    guard let classifier = classifier else {
      result(FlutterError(code: "NOT_INITIALIZED", message: "Call initialize() first", details: nil))
      return
    }

    guard
      let args       = call.arguments as? [String: Any],
      let yuvBytes   = args["yuvBytes"]   as? FlutterStandardTypedData,
      let width      = args["imageWidth"] as? Int,
      let height     = args["imageHeight"] as? Int,
      let bytesPerRow = args["bytesPerRow"] as? Int,
      let rectMap    = args["faceContour"] as? [String: Int],
      let rectLeft   = rectMap["left"],
      let rectTop    = rectMap["top"],
      let rectRight  = rectMap["right"],
      let rectBottom = rectMap["bottom"]
    else {
        result(FlutterError(code: "INVALID_ARGS", message: "Required: yuvBytes, width, height, rotation, rect{left,top,right,bottom}", details: nil))
        return
    }

    let startAt = Date()


    DispatchQueue.global(qos: .userInitiated).async {
        guard let bitmap = ImageUtils.bgraToUIImage(yuvBytes.data, width: width, height: height, bytesPerRow: bytesPerRow) else {
            DispatchQueue.main.async {
              result(FlutterError(code: "PREPROCESS_FAILED", message: "Could not decode image", details: nil))
            }
            return
        }
        
        
        let rect = CGRect(x: rectLeft, y: rectTop, width: rectRight - rectLeft, height: rectBottom - rectTop)
        let cropped = ImageUtils.cropBitmap(bitmap, rect: rect)

      guard let response = classifier.predict(image: cropped!) else {
        DispatchQueue.main.async {
          result(FlutterError(code: "INFERENCE_FAILED", message: "Model inference returned nil", details: nil))
        }
        return
      }

      let hasMask = response.mask > response.withoutMask

      DispatchQueue.main.async {
          result([
            "hasMask":          hasMask,
            "withMaskScore":    response.mask,
            "withoutMaskScore": response.withoutMask,
            "durationInMilliseconds": Date().timeIntervalSince(startAt) * 1000
          ])
      }
    }
  }
}




```
<br>
<br>


***mask_detector.dart***
```dart


class MaskDetector {
  
  static Future<Map<String, dynamic>> initialize() {
    return MaskDetectorPlatform.instance.initialize();
  }


  static Future<MaskDetectionResult> detect(Uint8List yuvBytes,{
    required double imageWidth,
    required double imageHeight,
    required Rect faceCountour,
    int rotation = 0,
  }) {
    return MaskDetectorPlatform.instance.detectMask(
      yuvBytes,
      imageHeight: imageHeight,
      imageWidth: imageWidth,
      faceCountour: faceCountour,
      rotation: rotation
    );
  }

  static Future<Map<String, dynamic>> destroy() {
    return MaskDetectorPlatform.instance.destroy();
  }

}


```

<br>
<br>

***mask_detection_view.dart***

```dart

class MaskDetectionView extends StatefulWidget {
  const MaskDetectionView({super.key});

  @override
  State<MaskDetectionView> createState() => _MaskDetectionViewState();
}

class _MaskDetectionViewState extends State<MaskDetectionView> {

  MaskDetectionResult? _maskResult;
  Size? _screenSize;
  late FaceDetector _faceDetector;
  bool isNoFaceDetected = true;
  bool _maskDetectorInitialized = false;
  CustomPaint? _customPaint;
  bool _isWidgetDestroyed = false;
  String? _processDuration;


  final GlobalKey<CameraViewState> _cameraViewKey = GlobalKey();

  void _handleDetectMask(CameraStreamPayload payload) async {
    
    if(_isWidgetDestroyed) return;
    
    final startAt = DateTime.now();
    try {

      final faces = await _faceDetector.processImage(payload.inputImage);
      if(faces.isEmpty) {
        isNoFaceDetected = true;
        throw Exception("No face detected");
      }

      isNoFaceDetected = false;

      final bx = faces[0].boundingBox;

      final faceContour = Rect.fromLTRB(bx.left, bx.top, bx.right,bx.bottom);

      final painter = FaceDetectorPainter(
        faces,
        Size(payload.cameraImage.width.toDouble(), payload.cameraImage.height.toDouble()),
        payload.inputImage.metadata!.rotation,
        CameraLensDirection.front
      );
      _customPaint = CustomPaint(painter: painter);

      _maskResult = await MaskDetector.detect(
        payload.yuvBytes,
        imageWidth: payload.imageWidth.toDouble(),
        imageHeight: payload.imageHeight.toDouble(),
        faceCountour: faceContour,
        rotation: payload.rotation
      );
      log("data : $_maskResult");
      _processDuration = "${DateTime.now().difference(startAt).inMilliseconds} ms";
    }
    catch (e){
      _maskResult = null;
      _processDuration = null;
      log("Error while detect mask : $e");
    } finally {
      setState(() {});
    }
  }

  void _initDetection() async {
    
    _faceDetector = FaceDetector(
      options: FaceDetectorOptions(
        performanceMode: FaceDetectorMode.accurate,
        minFaceSize: 0.3
      ),
    );

    final result = await MaskDetector.initialize();
    _maskDetectorInitialized = result['status'] ?? false;

    setState(() {});
  }

  @override
  void initState() {
    super.initState();
    _initDetection();
  }

  @override
  void dispose() {
    _isWidgetDestroyed = true;
    super.dispose();
    MaskDetector.destroy();
    _faceDetector.close();
  }


  @override
  Widget build(BuildContext context) {

    if(!_maskDetectorInitialized) {
      return Scaffold(
        body: Center(
          child: Text("Failed to initialize mask detector!."),
        ),
      );
    }

    _screenSize = MediaQuery.of(context).size;
    return Scaffold(
      appBar: AppBar(
        title: Text("Sample Mask Detection",style: TextStyle(color: Colors.white)),
        backgroundColor: Colors.blueAccent,
      ),
      backgroundColor: Color(0xFFC7D9E9),
      body: SafeArea(
        child: Column(
          children: [
            const SizedBox(height: 40),
            const Text(
              "Mask Detection",
              style: TextStyle(
                fontSize: 20,
                fontWeight: FontWeight.w600,
                color: Colors.blueGrey,
                letterSpacing: 2
              ),
            ),
            const SizedBox(height: 50),
            Center(
              child: SizedBox(
                width: _screenSize!.width * .9,
                height: _screenSize!.width * .9,
                child: CameraView(
                  key: _cameraViewKey,
                  onImage: _handleDetectMask,
                  customPaint: _customPaint,
                  cameraStreamProcessDelay: const Duration(milliseconds: 200),
                ),
              ),
            ),
            const SizedBox(height: 40),
            _buildResponse()
          ],
        ),
      ),
    );
  }
  Widget _buildResponse(){
    if(isNoFaceDetected){
      return Text(
        "No face detected!.",
        style: TextStyle(
          color: Colors.red,
          fontSize: 20
        ),
    );
    }
    if(_maskResult == null) return SizedBox.shrink();
    String msg = _maskResult!.hasMask ? "Has Mask" : "No Mask";
    Color color = _maskResult!.hasMask ? Colors.red : Colors.green;
    return Column(
      children: [
        Text(
          msg,
          style: TextStyle(
            color: color,
            fontSize: 20
          ),
        ),
        Text(
          "Confidence Score : ${_maskResult!.hasMask ? _maskResult!.withMaskScore : _maskResult!.withoutMaskScore}",
          style: TextStyle(
            color: color
          ),
        ),
        if(_processDuration != null)
          Text(
            "Duration : $_processDuration",
            style: TextStyle(color: Colors.green),
          )
      ],
    );
  }
}
```

<br>
<br>

## Conclusion
*In this blog, I aimed to share my exploration of Mask Detection, which can be helpful for KYC processes involving user face verification, such as Liveness Detection and Face Recognition.

<br>

*You can find full implement source code in repo :*

[https://github.com/horlengg/mask_detector](https://github.com/horlengg/mask_detector)

*Thank for reading!.*

<br>
<br>
<br>
<br>