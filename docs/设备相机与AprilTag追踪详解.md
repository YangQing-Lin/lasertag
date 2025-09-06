### 设备相机与 AprilTag 追踪详解

本文聚焦 `Assets/Anaglyph/XRTemplate/Camera` 与 `Assets/Anaglyph/XRTemplate/AprilTags`、`Assets/Anaglyph/XRTemplate/SharedSpaces/AprilTags`，解释设备相机数据流、AprilTag 是什么、在本项目中的作用与实现流程（检测→世界位姿→共位），并附代码引用。

---

## 什么是 AprilTag？

- AprilTag 是一种视觉 Fiducial 标记系统（类似于更鲁棒的 QR-like 标记），由黑白方框/内嵌编码组成。相机捕获图像后，可通过几何与编码解算得到每个标记的 ID 与相对相机的姿态（位置与旋转）。
- 特点：鲁棒、低光照可用、无主动发光；适合做室内定位、机器人/AR 框架的坐标系锚点。

在本项目中的作用：
- 作为共位（Colocation）的一种实现手段。当多个设备都能看到相同的物理 AprilTag，就可以用这些标签的“规范位置”（Canonical Positions）作为公共参考，计算设备 XR Rig 与场景坐标的齐次变换，从而实现多设备共享一个世界坐标系。

---

## 设备相机数据流（CameraReader）

`Assets/Anaglyph/XRTemplate/Camera/CameraReader.cs` (行 167-176)
```csharp
public async Task Configure(int index, int width, int height)
{
    if (DeviceIsOpen)
        throw new ConfiguredException("Cannot configure camera while camera is open!");
    if (!await CheckPermissions())
        throw new Exception("Camera does not have permission!");
    androidInterface.Configure(index, width, height);
    texture = new Texture2D(width, height, TextureFormat.R8, 1, false);
    bufferSize = width * height;
}
```

`Assets/Anaglyph/XRTemplate/Camera/CameraReader.cs` (行 283-293)
```csharp
private unsafe void OnImageAvailable()
{
    buffer = androidInterface.GetByteBuffer();
    if (buffer == default) return;
    texture.LoadRawTextureData((IntPtr)buffer, bufferSize);
    texture.Apply();
    TimestampNs = androidInterface.GetTimestamp();
    ImageAvailable.Invoke(texture);
}
```

要点：
- Android 原生接口拉取灰度图缓冲（R8），输出 `Texture2D` 与时间戳、硬件内外参；以事件 `ImageAvailable` 推送给上层（如 AprilTagTracker）。

---

## AprilTag 检测与世界位姿（AprilTagTracker）

`Assets/Anaglyph/XRTemplate/AprilTags/AprilTagTracker.cs` (行 68-89)
```csharp
while(enabled)
{
    await Awaitable.NextFrameAsync();
    if (!newFrameAvailable)
        continue;
    newFrameAvailable = false;
    if (detector == null)
        detector = new TagDetector(tex.width, tex.height, 1);
    var intrins = cameraReader.HardwareIntrinsics;
    var fov = 2 * Mathf.Atan((intrins.Resolution.y / 2f) / intrins.FocalLength.y);
    var size = tagSizeMeters;
    FrameTimestamp = cameraReader.TimestampNs * 0.000000001f;
    OVRPlugin.PoseStatef headPoseState = OVRPlugin.GetNodePoseStateAtTime(FrameTimestamp, OVRPlugin.Node.Head);
    var imgBytes = cameraReader.Texture.GetPixelData<byte>(0);
    await detector.Detect(imgBytes, fov, size);
```

`Assets/Anaglyph/XRTemplate/AprilTags/AprilTagTracker.cs` (行 100-111)
```csharp
foreach (var pose in detector.DetectedTags)
{
    TagPose worldPose = new(
        pose.ID,
        viewMat.MultiplyPoint(pose.Position),
        viewMat.rotation * pose.Rotation * Quaternion.Euler(-90, 0, 0));
    worldPoses.Add(worldPose);
}
OnDetectTags.Invoke(worldPoses);
```

要点：
- 通过硬件内参推算相机 FOV，调用原生 `TagDetector` 对灰度图进行检测，得到每个 Tag 的相机系下位姿。
- 结合头显瞬时位姿与相机相对位姿，将 Tag 坐标变换到 XR Rig 世界坐标（`MainXRRig.TrackingSpace.localToWorldMatrix`）。
- 最终以 `OnDetectTags` 事件输出 `TagPose` 列表（包含 ID、世界空间位置与旋转）。

---

## 用 AprilTag 做共位（AprilTagColocator）

`Assets/Anaglyph/XRTemplate/SharedSpaces/AprilTags/AprilTagColocator.cs` (行 53-64)
```csharp
[Rpc(SendTo.Everyone)]
private void RegisterCanonTagRpc(int id, Vector3 canonicalPosition)
{
    canonTags[id] = canonicalPosition;
}
[Rpc(SendTo.SpecifiedInParams)]
private void SyncCanonTagsRpc(int[] id, Vector3[] positions, RpcParams rpcParams = default)
{
    for (int i = 0; i < id.Length; i++)
        canonTags[id[i]] = positions[i];
}
```

`Assets/Anaglyph/XRTemplate/SharedSpaces/AprilTags/AprilTagColocator.cs` (行 147-156)
```csharp
Matrix4x4 trackingSpace = MainXRRig.TrackingSpace.localToWorldMatrix;
Matrix4x4 delta = IterativeClosestPoint.FitCorresponding(
    sharedLocalPositions.ToArray(), trackingSpace,
    sharedCanonPositions.ToArray(), float4x4.identity);
trackingSpace = delta * trackingSpace;
MainXRRig.TrackingSpace.position = trackingSpace.GetPosition();
MainXRRig.TrackingSpace.rotation = trackingSpace.rotation;
IsColocated = true;
```

`Assets/Anaglyph/XRTemplate/SharedSpaces/AprilTags/AprilTagColocator.cs` (行 195-215)
```csharp
if (IsOwner)
{
    var headState = OVRPlugin.GetNodePoseStateAtTime(tagTracker.FrameTimestamp, OVRPlugin.Node.Head);
    var v = headState.Velocity; Vector3 vel = new(v.x, v.y, v.z);
    float headSpeed = vel.magnitude;
    var av = headState.AngularVelocity; Vector3 angVel = new(av.x, av.y, av.z);
    float angHeadSpeed = angVel.magnitude;
    bool headIsStable = headSpeed < maxHeadSpeed && angHeadSpeed < maxHeadAngSpeed;
    if (headIsStable)
    {
        bool locked = tagsLocked.Contains(result.ID);
        bool isCloseEnough = TagIsWithinRegisterDistance(globalPos);
        if (!locked && isCloseEnough)
        {
            RegisterCanonTagRpc(result.ID, globalPos);
        }
    }
}
```

要点：
- Owner 端维护一份“规范标签位置”表（`canonTags`），通过 RPC 同步到新连入的客户端；每帧收集“本地观测的标签位置”（`localTags`）。
- 找到两边都可见的标签集合（共享 ID），将本地位置与规范位置成对输入到 `ICP` 以计算变换 `delta`，并累计应用到 `MainXRRig.TrackingSpace`，从而将设备对齐到共享坐标。
- 为避免错误注册，Owner 仅在“头部速度/角速度稳定”且“与头部足够近”时才把观测到的标签坐标注册为规范位置。

---

## 与 Meta Anchors 的对比

- 两者都是共位方案：
  - Meta Anchors：依赖 `OVRSpatialAnchor` 的本地化与分享，使用硬件空间锚生态；
  - AprilTag：依赖纸质/实体标签，无需硬件 Anchor；适合临时/布标场景，配置灵活。
- 在项目中，可通过 `LaserTag/ColocationManager` 切换使用方式。

---

## 小结
- 设备相机模块提供图像与时序、相机内外参，供 AprilTag 检测使用。
- AprilTagTracker 将检测结果转换为世界坐标 `TagPose` 列表。
- AprilTagColocator 利用“本地观测标签位置 vs 规范标签位置”做 ICP 对齐，并用 RPC 维护规范表，实现多人共位。


