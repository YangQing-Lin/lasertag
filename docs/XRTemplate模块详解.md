### XRTemplate 模块详解（XR 框架、环境深度、共位）

本文聚焦 `Assets/Anaglyph/XRTemplate`，覆盖 XR Rig 与手性标注、环境深度（DepthKit/EnvironmentMapper）、设备相机（CameraReader）、共位接口与实现（Meta Anchors、AprilTag 参考）等核心逻辑。附代码引用采用“路径 + 行号 + csharp 代码块”的格式，便于工程定位。

---

## XR Rig 与手性

`Assets/Anaglyph/XRTemplate/MainXRRig.cs` (行 27-47)
```csharp
public static void MatchPoseToTarget(Pose current, Pose target, bool enforceUp = true, bool flattenUp = false)
{
    var root = MainXRRig.TrackingSpace;
    if (enforceUp)
    {
        current = FlattenPoseRotation(current, flattenUp);
        target = FlattenPoseRotation(target, flattenUp);
    }
    Matrix4x4 rigMat = Matrix4x4.TRS(root.position, root.rotation, Vector3.one);
    Matrix4x4 localMat = Matrix4x4.TRS(current.position, current.rotation, Vector3.one);
    Matrix4x4 targetMat = Matrix4x4.TRS(target.position, target.rotation, Vector3.one);
    Matrix4x4 rigLocalToAnchor = localMat.inverse * rigMat;
    Matrix4x4 relativeToDesired = targetMat * rigLocalToAnchor;
    root.SetPositionAndRotation(relativeToDesired.GetPosition(), relativeToDesired.rotation);
}
```

`Assets/Anaglyph/XRTemplate/HandedHierarchy.cs` (行 20-31)
```csharp
public InteractorHandedness Handedness
{
    get => _handedness;
    set
    {
        if (_handedness != value)
        {
            _handedness = value;
            UpdateNode();
            OnHandednessChange?.Invoke(value);
        }
    }
}
```

---

## 环境深度：DepthKit 驱动与体素化映射

### DepthKit 驱动（读取环境深度与法线、设置全局矩阵）

`Assets/Anaglyph/XRTemplate/Depth/DepthKitDriver.cs` (行 60-78)
```csharp
public void UpdateCurrentRenderingState()
{
    Texture depthTex = Shader.GetGlobalTexture(Meta_EnvironmentDepthTexture_ID);
    DepthAvailable = depthTex != null;
    if (!DepthAvailable)
        return;
    Shader.SetGlobalVector(agDepthTexSize, new Vector2(depthTex.width, depthTex.height));
    Shader.SetGlobalTexture(agDepthTex_ID, depthTex);
    Shader.SetGlobalTexture(agDepthEdgeTex_ID,
        Shader.GetGlobalTexture(Meta_PreprocessedEnvironmentDepthTexture_ID));
    Shader.SetGlobalVector(agDepthZParams_ID,
        Shader.GetGlobalVector(Meta_EnvironmentDepthZBufferParams_ID));
```

`Assets/Anaglyph/XRTemplate/Depth/DepthKitDriver.cs` (行 100-109)
```csharp
for (int i = 0; i < agDepthProj.Length; i++)
{
    var desc = Utils.GetEnvironmentDepthFrameDesc(i);
    agDepthProj[i] = CalculateDepthProjMatrix(desc);
    agDepthProjInv[i] = Matrix4x4.Inverse(agDepthProj[i]);
    agDepthView[i] = CalculateDepthViewMatrix(desc) * MainXRRig.TrackingSpace.worldToLocalMatrix;
    agDepthViewInv[i] = Matrix4x4.Inverse(agDepthView[i]);
}
```

### EnvironmentMapper（体素集成与射线）

`Assets/Anaglyph/XRTemplate/Depth/EnvironmentMapper.cs` (行 90-108)
```csharp
private async void ScanLoop()
{
    while (enabled)
    {
        await Awaitable.WaitForSecondsAsync(1f / dispatchesPerSecond);
        var depthTex = Shader.GetGlobalTexture(depthTexID);
        if (depthTex == null) continue;
        var normTex = Shader.GetGlobalTexture(normTexID);
        if (frustumVolume == null)
            Setup();
        Matrix4x4 view = Shader.GetGlobalMatrixArray(viewID)[0];
        Matrix4x4 proj = Shader.GetGlobalMatrixArray(projID)[0];
        ApplyScan(depthTex, normTex, view, proj);
    }
}
```

`Assets/Anaglyph/XRTemplate/Depth/EnvironmentMapper.cs` (行 198-206)
```csharp
public static bool Raycast(Ray ray, float maxDist, out RayResult result, bool fallback = false)
    => Instance.RaycastInternal(ray, maxDist, out result, fallback);
```

`Assets/Anaglyph/XRTemplate/Depth/EnvironmentMapper.cs` (行 221-236)
```csharp
shader.SetVector("rcOrig", ray.origin);
shader.SetVector("rcDir", ray.direction);
shader.SetFloat("rcIntScale", RaycastScaleFactor);
uint lengthInt = (uint)(maxDist * RaycastScaleFactor);
ComputeBuffer resultBuffer = new ComputeBuffer(1, sizeof(uint));
resultBuffer.SetData(new uint[] { lengthInt });
raycastKernel.Set("rcResult", resultBuffer);
int totalNumSteps = Mathf.RoundToInt(maxDist / metersPerVoxel);
if (totalNumSteps == 0)
    return false;
raycastKernel.DispatchGroups(totalNumSteps, 1, 1);
```

---

## 设备相机：CameraReader（Meta/Android）

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
    // 读取设备坐标系下的相机位姿/内参，转换为世界系使用
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

---

## 共位（SharedSpaces）

### 共位接口与管理器

`Assets/Anaglyph/XRTemplate/SharedSpaces/Colocation.cs` (行 5-12)
```csharp
public interface IColocator
{
    public bool IsColocated { get; }
    public event Action<bool> IsColocatedChange;
    public void Colocate();
    public void StopColocation();
}
```

`Assets/Anaglyph/XRTemplate/SharedSpaces/Colocation.cs` (行 34-48)
```csharp
public static void SetActiveColocator(IColocator colocator)
{
    if (activeColocator != null)
    {
        activeColocator.IsColocatedChange -= SetIsColocated;
    }
    activeColocator = colocator;
    if (activeColocator != null)
    {
        IsColocated = activeColocator.IsColocated;
        activeColocator.IsColocatedChange += SetIsColocated;
    }
}
```

### Meta Anchors 共位流程

`Assets/Anaglyph/XRTemplate/SharedSpaces/MetaAnchors/MetaAnchorColocator.cs` (行 36-56)
```csharp
public void InstantiateNewAnchor()
{
    if(ColocationAnchor.Instance != null)
        ColocationAnchor.Instance.DespawnAndDestroyRpc();
    Transform head = MainXRRig.Camera.transform;
    Vector3 spawnPos = head.position;
    spawnPos.y -= 1.5f;
    Vector3 flatForward = head.transform.forward; flatForward.y = 0; flatForward.Normalize();
    Quaternion spawnRot = Quaternion.LookRotation(flatForward, Vector3.up);
    GameObject g = Instantiate(anchorPrefab.gameObject, spawnPos, spawnRot);
    g.TryGetComponent(out NetworkObject networkObject);
    networkObject.Spawn();
}
```

`Assets/Anaglyph/XRTemplate/SharedSpaces/MetaAnchors/MetaAnchorColocator.cs` (行 113-118)
```csharp
if (anchor.IsAnchored)
{
    Pose anchorPose = anchor.transform.GetWorldPose();
    MainXRRig.MatchPoseToTarget(anchorPose, anchor.DesiredPose);
    IsColocated = true;
}
```

### NetworkedAnchor（锚点网络化与共享下载）

`Assets/Anaglyph/XRTemplate/SharedSpaces/MetaAnchors/NetworkedAnchor.cs` (行 103-118)
```csharp
public async Task Share()
{
    if (!IsOwner)
        throw new Exception("Only the anchor owner can share it!");
    OriginalPoseSync.Value = new NetworkPose(transform);
    spatialAnchor.enabled = true;
    bool localizeSuccess = await spatialAnchor.WhenLocalizedAsync();
    if (!localizeSuccess)
        throw new NetworkedAnchorException($"Failed to localize anchor {spatialAnchor.Uuid}");
    anchored = true;
    var shareResult = await spatialAnchor.ShareAsync(spatialAnchor.Uuid);
    if (!shareResult.Success)
        throw new NetworkedAnchorException($"Failed to share anchor {spatialAnchor.Uuid}: {shareResult}");
    Uuid.Value = new(spatialAnchor.Uuid);
}
```

`Assets/Anaglyph/XRTemplate/SharedSpaces/MetaAnchors/NetworkedAnchor.cs` (行 224-261)
```csharp
private async Task Load(Guid uuid)
{
    List<UnboundAnchor> loadedAnchors = new();
    var downloadResult = await LoadUnboundSharedAnchorsAsync(uuid, loadedAnchors);
    if (!downloadResult.Success)
        throw new NetworkedAnchorException($"Failed to download anchor {uuid}: {downloadResult}");
    var unboundAnchor = loadedAnchors[0];
    await LocalizeAndBindAsync(unboundAnchor);
}
```

---

## 小结
- MainXRRig 提供 Rig 与目标姿态的对齐能力，支持“保持向上/扁平化 Up”。
- DepthKitDriver 读取 Meta 环境深度与 Z 参数，输出视图/投影/法线至全局 Shader；EnvironmentMapper 将其体素化、支持 GPU 射线命中。
- CameraReader 负责设备相机权限、配置与数据回调，供 AprilTag 等上层功能使用。
- 共位接口统一管理当前 Colocator，Meta Anchors 实现通过网络锚共享/下载并用 Rig 对齐实现空间一致。


