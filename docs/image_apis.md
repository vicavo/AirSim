# Image APIs

## Getting Single Image
Here's a sample code to get a single image:

```
int getOneImage() 
{
    using namespace std;
    using namespace msr::airlib;
    
    msr::airlib::RpcLibClient client;

    vector<uint8_t> png_image = client.simGetImage(0, DroneControlBase::ImageType::DepthVis);
    //do something with images
}
```

Returned image is always in png format. To get uncompressed and other format please see next section.

## Getting Stereo/Multiple Images at Once

The `simGetImages` API which is slightly more complex to use than `simGetImage` API, for example, you can get left camera view, right camera view and depth image from left camera - all at once! 

```
int getStereoAndDepthImages() 
{
    using namespace std;
    using namespace msr::airlib;
    
    typedef DroneControllerBase::ImageRequest ImageRequest;
    typedef VehicleCameraBase::ImageResponse ImageResponse;
    typedef VehicleCameraBase::ImageType_ ImageType_;

    msr::airlib::RpcLibClient client;

    //get right, left and depth images. First two as png, second as float16.
    vector<ImageRequest> request = { 
        ImageRequest(0, ImageType_::Scene), 
        ImageRequest(1, ImageType_::Scene),        
        ImageRequest(1, ImageType_::DepthMeters, true) 
    };
    const vector<ImageResponse>& response = client.simGetImages(request);
    //do something with response which contains image data, pose, timestamp etc
}
```
For a ready to run sample code please see [sample code in HelloDrone project](../HelloDrone/main.cpp). 

See also [complete code](../Examples/StereoImageGenerator.hpp) that generates specified number of stereo images and ground truth depth with normalization to camera plan, computation of disparity image and saving it to pfm format.

Unlike `simGetImage`, the `simGetImages` API also allows you to get uncompressed images as well as floating point single channel images (instead of 3 channel (RGB), each 8 bit).

You can also use Python to get images. For sample code please see [PythonClient project](https://github.com/Microsoft/AirSim/tree/master/PythonClient) and [Python example doc](python.md).

## "Computer Vision" Mode

You can use AirSim in so-called "Computer Vision" mode. In this mode, physics engine is disabled and there is no flight controller active. This means when you start AirSim, vehicle would just hang in the air. However you can move around using keyboard (use F1 to see help on keys). You can press Record button to continuously generate images. Or you can call APIs to move around and take images.

To active this mode, edit [settings.json](settings.json) that you can find in your `Documents\AirSim` folder (or `~/Documents/AirSim` on Linux) and make sure following values exist at root level:

```
{
  "UsageScenario": "ComputerVision"
}
```

If you are only interested in this mode, you might also want to take a look at [UnrealCV project](http://unrealcv.org/).

## How to Set Position and Orientation (Pose)?

To move around the environment using APIs you can use `simSetPose` API. This API takes position and orientation and sets that on the vehicle. If you don't want to change position (or orientation) then set components of position (or orientation) to floating point nan values.

## Changing Resolution and Camera Parameters
To change resolution, FOV etc, you can use [settings.json](settings.md). For example, below is the complete content of settings.json that sets parameters for scene capture and uses "Computer Vision" mode described above. If you omit any setting then below default values will be used. For more information see [settings doc](settings.md). If you are using stereo camera, currently the distance between left and right is fixed at 25 cm.

```
{
  "CaptureSettings": [
    {
      "ImageType": 0,
      "Width": 256,
      "Height": 144,
      "FOV_Degrees": 90,
      "AutoExposureSpeed": 100,
      "MotionBlurAmount": 0
    }
  ],
  "UsageScenario": "ComputerVision"
}
```

## What Does Pixel Values Mean in Different Image Types?
### Depth Image
You normally want retrieve depth image as float (i.e. set `pixels_as_float = true` and specify `ImageType = DepthMeters` in `ImageRequest`) in which case each pixel is depth in *camera plane* in meters. This way ground truth depth image is directly compatible with estimated depth image generated by stereo algorithms. There is no need to convert depth to planner value like in old versions.

### Depth Visualization Image
When you specify `ImageType = DepthVis` in `ImageRequest`, you get image that helps depth visualization. In this case, each pixel value is interpolated from red to green depending on depth in camera plane in meters. The pixels with pure green means depth of 100m or more while pure red means depth of 0 meters.

### Disparity Image
You normally want retrieve disparity image as float (i.e. set `pixels_as_float = true` and specify `ImageType = DisparityNormalized` in `ImageRequest`) in which case each pixel is `(Xl - Xr)/Xmax`, valued from 0 to 1.

### Segmentation Image
When you specify `ImageType = Segmentation` in `ImageRequest`, you get image that gives you ground truth segmentation of the scene. AirSim assigns value 0 to 255 to each mesh available in environment. This value is than mapped to a specific color in [the pallet](../Unreal/Plugins/AirSim/Content/HUDAssets/seg_color_pallet.png). Currently, if you had like to assign specific value to specific map, you will need to [change code](../Unreal/Plugins/AirSim/Source/FlyingPawn.cpp#L28). We are planning to enable APIs for this in future.

## Collision API
The collision information can be obtained using `getCollisionInfo` API. This call returns a struct that has information not only whether collision occurred but also collision position, surface normal, penetration depth and so on.

## Example Code
A complete example of setting vehicle positions at random locations and orientations and then taking images can be found in [GenerateImageGenerator.hpp](../Examples/StereoImageGenerator.hpp). This example generates specified number of stereo images and ground truth disparity image and saving it to pfm format.
