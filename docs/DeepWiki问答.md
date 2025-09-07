# 项目 DeepWiki 地址

https://deepwiki.com/anaglyphs/lasertag



# DeepWiki Q&A with Code Context for Repository: anaglyphs/lasertag


## Q1

有关下面的内容：

5.1 AprilTag Colocation
The AprilTagColocator implements the IColocator interface and uses camera-based tracking of AprilTag markers to align coordinate systems:

Camera capture starts via CameraManager.Instance.StartCapture()
AprilTagTracker processes frames and detects tags
Detected tags are shared over the network with their positions
The client aligns its tracking space to match the host's reference frame
When alignment is successful, the colocator sets IsColocated = true, which triggers the connection state to update to "Connected".

是否可以不使用额外的规范化显示物品，而是直接扫描当前场景，从中找出合适的锚点进行定位？


### Answer

## Answer

您询问的是关于 AprilTag 定位系统是否可以不使用额外的标准化显示物品（AprilTag 标记），而是直接扫描当前场景来找出合适的锚点进行定位。<cite/>

从代码库来看，当前的 `AprilTagColocator` 确实依赖于物理 AprilTag 标记来进行定位。 lasertag:36-44  系统通过 `CameraManager` 启动相机捕获，然后 `AprilTagTracker` 处理帧数据来检测标记。 lasertag:82-90 

但是，代码库中已经存在其他不需要额外物理标记的定位方法：

1. **Meta Anchor 方法**：`MetaAnchorColocator` 使用 Meta 的空间锚点系统，可以在场景中创建虚拟锚点。 lasertag:34-50  这个方法会在头部位置附近的地面创建一个锚点，不需要任何物理标记。 lasertag:37-45 

2. **键盘追踪方法**：系统还支持使用物理键盘作为参考点的 `MetaTrackableColocator`。 lasertag:53-55 

`ColocationManager` 允许在这些方法之间切换。 lasertag:47-62  默认情况下使用 `Automatic` 方法（Meta Anchor），只有在启用键盘定位时才会使用 `TrackedKeyboard` 方法。 lasertag:41-45 

因此，您完全可以使用 Meta Anchor 方法来实现无需额外物理标记的场景扫描定位，这已经是系统的默认行为。<cite/>

## Notes

Meta Anchor 方法利用 Meta Quest 的内置空间理解能力来创建和共享虚拟锚点，这正是您所寻求的"直接扫描当前场景"的解决方案。AprilTag 方法主要用于需要精确控制参考点位置或在 Meta 空间锚点不可用的情况下的备选方案。

Wiki pages you might want to explore:
- [Project Configuration (anaglyphs/lasertag)](/wiki/anaglyphs/lasertag#1.1)
- [Colocation System (anaglyphs/lasertag)](/wiki/anaglyphs/lasertag#2.3)


### Citations


**File:** Assets/Anaglyph/XRTemplate/SharedSpaces/AprilTags/AprilTagColocator.cs (L36-44)

```csharp
		public void Colocate()
		{
			IsColocated = false;
			colocationActive = true;

			CameraManager.Instance.Configure(0, 320, 240);
			CameraManager.Instance.StartCapture();
			AprilTagTracker.Instance.OnDetectTags += OnDetectTags;
		}
```

**File:** Assets/Anaglyph/XRTemplate/Camera/AprilTags/AprilTagTracker.cs (L82-90)

```csharp
				if(detector == null)
					detector = new TagDetector(tex.width, tex.height, 1);

				var intrins = CameraManager.Instance.CamIntrinsics;
				var fov = 2 * Mathf.Atan((intrins.Resolution.y / 2f) / intrins.FocalLength.y);
				var size = tagSizeMeters;

				var imgBytes = CameraManager.Instance.CamTex.GetPixelData<byte>(0);
				await detector.SchedulePoseEstimationJob(imgBytes, fov, size);
```

**File:** Assets/Anaglyph/XRTemplate/SharedSpaces/MetaAnchors/MetaAnchorColocator.cs (L34-50)

```csharp
			if (allAnchors.Length == 0)
			{
				// spawn anchor
				Transform head = MainXROrigin.Instance.Camera.transform;

				Vector3 spawnPos = head.position;
				spawnPos.y = 0;

				Vector3 flatForward = head.transform.forward;
				flatForward.y = 0;
				flatForward.Normalize();
				Quaternion spawnRot = Quaternion.LookRotation(flatForward, Vector3.up);

				GameObject prefab = Resources.Load<GameObject>("Networked Colocation Anchor");
				GameObject g = Instantiate(prefab, spawnPos, spawnRot);
				g.TryGetComponent(out NetworkedAnchor networkedAnchor);
				networkedAnchor.NetworkObject.Spawn();
```

**File:** Assets/Anaglyph/LaserTag/ColocationManager.cs (L41-45)

```csharp
			if (IsOwner)
			{
				colocationMethodSync.Value = useKeyboardColocation.Value ?
					ColocationMethod.TrackedKeyboard : ColocationMethod.Automatic;
			}
```

**File:** Assets/Anaglyph/LaserTag/ColocationManager.cs (L47-62)

```csharp
			switch (colocationMethodSync.Value)
			{
				case ColocationMethod.Automatic:
					Colocation.SetActiveColocator(metaAnchorColocator);
					break;

				case ColocationMethod.TrackedKeyboard:
					Colocation.SetActiveColocator(metaTrackableColocator);
					break;

				case ColocationMethod.AprilTag:
					Colocation.SetActiveColocator(aprilTagColocator);
					break;
			}

			Colocation.ActiveColocator.Colocate();
```

## Q2

关于项目的网络连接部分，是否需要所有的XR设备都连入同一个路由器或交换机？


### Answer

## Answer

您询问的是关于项目网络连接部分，是否需要所有XR设备都连入同一个路由器或交换机。<cite/>

根据代码库分析，该项目支持两种网络连接协议，**不一定**需要所有设备连入同一个路由器：<cite/>

### 1. 本地网络连接 (LAN)
当使用本地网络模式时，设备确实需要在同一个网络中。 lasertag:30-45  系统会获取本地IP地址并在指定端口（默认7777）上建立连接。 lasertag:282-285 

### 2. Unity Relay 服务连接
项目还支持通过Unity的中继服务器进行连接，这种方式**不需要**设备在同一个本地网络中。 lasertag:48-67  设置界面中提到"企业网络可能会导致问题，通过中继服务器路由连接可以解决这个问题"。 lasertag:474-477 

### 自动发现机制
`AutomaticNetworkConnector` 实现了自动发现功能，使用Meta Quest的`OVRColocationSession`来发现附近的设备。 lasertag:31-33  系统会根据当前使用的传输协议（本地网络或中继）来广播相应的连接信息。 lasertag:95-112 

用户可以在多人游戏菜单中选择使用Unity Relay服务。 lasertag:162-165  当启用中继服务时，设备可以通过互联网连接，无需在同一个本地网络中。<cite/>

## Notes

总结：如果使用本地网络模式，设备需要在同一个路由器/交换机下；如果使用Unity Relay服务，设备可以通过互联网连接，不受本地网络限制。项目的自动发现功能会根据选择的连接方式来处理设备发现和连接。

Wiki pages you might want to explore:
- [Colocation System (anaglyphs/lasertag)](/wiki/anaglyphs/lasertag#2.3)


### Citations


**File:** Assets/Anaglyph/Netcode/NetworkManagerHelper.cs (L30-45)

```csharp
				case Protocol.LAN:

					string localAddress = "";
					var addresses = Dns.GetHostEntry(Dns.GetHostName()).AddressList;
					foreach (var address in addresses)
					{
						if (address.AddressFamily == System.Net.Sockets.AddressFamily.InterNetwork)
						{
							localAddress = address.ToString();
							break;
						}
					}

					transport.SetConnectionData(localAddress, port);
					StartHost();
					break;
```

**File:** Assets/Anaglyph/Netcode/NetworkManagerHelper.cs (L48-67)

```csharp
				case Protocol.UnityService:

					await SetupServices();


					//Allocation allocation = await RelayService.Instance.CreateAllocationAsync(20);
					//allocationId = allocation.AllocationId;

					//transport.SetRelayServerData(allocation.ToRelayServerData(contyp));
					//StartHost();

					var options = new SessionOptions()
					{
						Name = Guid.NewGuid().ToString(),
						MaxPlayers = 20,
					}.WithDistributedAuthorityNetwork();

					await MultiplayerService.Instance.CreateSessionAsync(options);

					break;
```

**File:** Assets/Anaglyph/LaserTag/Systems.prefab (L282-285)

```text
  ConnectionData:
    Address: 127.0.0.1
    Port: 7777
    ServerListenAddress: 0.0.0.0
```

**File:** Assets/Anaglyph/LaserTag/Interface/Menu/Settings.prefab (L474-477)

```text
      value: By default, headsets try to connect over the local network, but enterprise
        networks can cause issues. Routing connections through a relay server can
        this.
      objectReference: {fileID: 0}
```

**File:** Assets/Anaglyph/XRTemplate/SharedSpaces/MetaAnchors/AutomaticNetworkConnector.cs (L31-33)

```csharp
			OVRColocationSession.ColocationSessionDiscovered += HandleColocationSessionDiscovered;

			HandleChange();
```

**File:** Assets/Anaglyph/XRTemplate/SharedSpaces/MetaAnchors/AutomaticNetworkConnector.cs (L95-112)

```csharp
			switch (transport.Protocol)
			{
				case UnityTransport.ProtocolType.UnityTransport:

					string address = transport.ConnectionData.Address;
					message = LanPrefix + address;
					break;

				case UnityTransport.ProtocolType.RelayUnityTransport:

					var sessionIds = await MultiplayerService.Instance.GetJoinedSessionIdsAsync();
					message = RelayPrefix + sessionIds[sessionIds.Count - 1];

					break;
			}

			Log($"Starting advertisement {message}");
			await OVRColocationSession.StartAdvertisementAsync(Encoding.ASCII.GetBytes(message));
```

**File:** Assets/Anaglyph/LaserTag/Interface/Menu/MultiplayerMenu.cs (L162-165)

```csharp
			var service = useUnityRelayService.Value ? NetworkHelper.Protocol.UnityService : NetworkHelper.Protocol.LAN;

			NetworkHelper.Host(service);
		}
```





# DeepWiki Q&A with Code Context for Repository: anaglyphs/lasertag


## Q1

有关下面的内容：

5.1 AprilTag Colocation
The AprilTagColocator implements the IColocator interface and uses camera-based tracking of AprilTag markers to align coordinate systems:

Camera capture starts via CameraManager.Instance.StartCapture()
AprilTagTracker processes frames and detects tags
Detected tags are shared over the network with their positions
The client aligns its tracking space to match the host's reference frame
When alignment is successful, the colocator sets IsColocated = true, which triggers the connection state to update to "Connected".

是否可以不使用额外的规范化显示物品，而是直接扫描当前场景，从中找出合适的锚点进行定位？


### Answer

## Answer

您询问的是关于 AprilTag 定位系统是否可以不使用额外的标准化显示物品（AprilTag 标记），而是直接扫描当前场景来找出合适的锚点进行定位。<cite/>

从代码库来看，当前的 `AprilTagColocator` 确实依赖于物理 AprilTag 标记来进行定位。 lasertag:36-44  系统通过 `CameraManager` 启动相机捕获，然后 `AprilTagTracker` 处理帧数据来检测标记。 lasertag:82-90 

但是，代码库中已经存在其他不需要额外物理标记的定位方法：

1. **Meta Anchor 方法**：`MetaAnchorColocator` 使用 Meta 的空间锚点系统，可以在场景中创建虚拟锚点。 lasertag:34-50  这个方法会在头部位置附近的地面创建一个锚点，不需要任何物理标记。 lasertag:37-45 

2. **键盘追踪方法**：系统还支持使用物理键盘作为参考点的 `MetaTrackableColocator`。 lasertag:53-55 

`ColocationManager` 允许在这些方法之间切换。 lasertag:47-62  默认情况下使用 `Automatic` 方法（Meta Anchor），只有在启用键盘定位时才会使用 `TrackedKeyboard` 方法。 lasertag:41-45 

因此，您完全可以使用 Meta Anchor 方法来实现无需额外物理标记的场景扫描定位，这已经是系统的默认行为。<cite/>

## Notes

Meta Anchor 方法利用 Meta Quest 的内置空间理解能力来创建和共享虚拟锚点，这正是您所寻求的"直接扫描当前场景"的解决方案。AprilTag 方法主要用于需要精确控制参考点位置或在 Meta 空间锚点不可用的情况下的备选方案。

Wiki pages you might want to explore:
- [Project Configuration (anaglyphs/lasertag)](/wiki/anaglyphs/lasertag#1.1)
- [Colocation System (anaglyphs/lasertag)](/wiki/anaglyphs/lasertag#2.3)


### Citations


**File:** Assets/Anaglyph/XRTemplate/SharedSpaces/AprilTags/AprilTagColocator.cs (L36-44)

```csharp
		public void Colocate()
		{
			IsColocated = false;
			colocationActive = true;

			CameraManager.Instance.Configure(0, 320, 240);
			CameraManager.Instance.StartCapture();
			AprilTagTracker.Instance.OnDetectTags += OnDetectTags;
		}
```

**File:** Assets/Anaglyph/XRTemplate/Camera/AprilTags/AprilTagTracker.cs (L82-90)

```csharp
				if(detector == null)
					detector = new TagDetector(tex.width, tex.height, 1);

				var intrins = CameraManager.Instance.CamIntrinsics;
				var fov = 2 * Mathf.Atan((intrins.Resolution.y / 2f) / intrins.FocalLength.y);
				var size = tagSizeMeters;

				var imgBytes = CameraManager.Instance.CamTex.GetPixelData<byte>(0);
				await detector.SchedulePoseEstimationJob(imgBytes, fov, size);
```

**File:** Assets/Anaglyph/XRTemplate/SharedSpaces/MetaAnchors/MetaAnchorColocator.cs (L34-50)

```csharp
			if (allAnchors.Length == 0)
			{
				// spawn anchor
				Transform head = MainXROrigin.Instance.Camera.transform;

				Vector3 spawnPos = head.position;
				spawnPos.y = 0;

				Vector3 flatForward = head.transform.forward;
				flatForward.y = 0;
				flatForward.Normalize();
				Quaternion spawnRot = Quaternion.LookRotation(flatForward, Vector3.up);

				GameObject prefab = Resources.Load<GameObject>("Networked Colocation Anchor");
				GameObject g = Instantiate(prefab, spawnPos, spawnRot);
				g.TryGetComponent(out NetworkedAnchor networkedAnchor);
				networkedAnchor.NetworkObject.Spawn();
```

**File:** Assets/Anaglyph/LaserTag/ColocationManager.cs (L41-45)

```csharp
			if (IsOwner)
			{
				colocationMethodSync.Value = useKeyboardColocation.Value ?
					ColocationMethod.TrackedKeyboard : ColocationMethod.Automatic;
			}
```

**File:** Assets/Anaglyph/LaserTag/ColocationManager.cs (L47-62)

```csharp
			switch (colocationMethodSync.Value)
			{
				case ColocationMethod.Automatic:
					Colocation.SetActiveColocator(metaAnchorColocator);
					break;

				case ColocationMethod.TrackedKeyboard:
					Colocation.SetActiveColocator(metaTrackableColocator);
					break;

				case ColocationMethod.AprilTag:
					Colocation.SetActiveColocator(aprilTagColocator);
					break;
			}

			Colocation.ActiveColocator.Colocate();
```

## Q2

关于项目的网络连接部分，是否需要所有的XR设备都连入同一个路由器或交换机？


### Answer

## Answer

您询问的是关于项目网络连接部分，是否需要所有XR设备都连入同一个路由器或交换机。<cite/>

根据代码库分析，该项目支持两种网络连接协议，**不一定**需要所有设备连入同一个路由器：<cite/>

### 1. 本地网络连接 (LAN)
当使用本地网络模式时，设备确实需要在同一个网络中。 lasertag:30-45  系统会获取本地IP地址并在指定端口（默认7777）上建立连接。 lasertag:282-285 

### 2. Unity Relay 服务连接
项目还支持通过Unity的中继服务器进行连接，这种方式**不需要**设备在同一个本地网络中。 lasertag:48-67  设置界面中提到"企业网络可能会导致问题，通过中继服务器路由连接可以解决这个问题"。 lasertag:474-477 

### 自动发现机制
`AutomaticNetworkConnector` 实现了自动发现功能，使用Meta Quest的`OVRColocationSession`来发现附近的设备。 lasertag:31-33  系统会根据当前使用的传输协议（本地网络或中继）来广播相应的连接信息。 lasertag:95-112 

用户可以在多人游戏菜单中选择使用Unity Relay服务。 lasertag:162-165  当启用中继服务时，设备可以通过互联网连接，无需在同一个本地网络中。<cite/>

## Notes

总结：如果使用本地网络模式，设备需要在同一个路由器/交换机下；如果使用Unity Relay服务，设备可以通过互联网连接，不受本地网络限制。项目的自动发现功能会根据选择的连接方式来处理设备发现和连接。

Wiki pages you might want to explore:
- [Colocation System (anaglyphs/lasertag)](/wiki/anaglyphs/lasertag#2.3)


### Citations


**File:** Assets/Anaglyph/Netcode/NetworkManagerHelper.cs (L30-45)

```csharp
				case Protocol.LAN:

					string localAddress = "";
					var addresses = Dns.GetHostEntry(Dns.GetHostName()).AddressList;
					foreach (var address in addresses)
					{
						if (address.AddressFamily == System.Net.Sockets.AddressFamily.InterNetwork)
						{
							localAddress = address.ToString();
							break;
						}
					}

					transport.SetConnectionData(localAddress, port);
					StartHost();
					break;
```

**File:** Assets/Anaglyph/Netcode/NetworkManagerHelper.cs (L48-67)

```csharp
				case Protocol.UnityService:

					await SetupServices();


					//Allocation allocation = await RelayService.Instance.CreateAllocationAsync(20);
					//allocationId = allocation.AllocationId;

					//transport.SetRelayServerData(allocation.ToRelayServerData(contyp));
					//StartHost();

					var options = new SessionOptions()
					{
						Name = Guid.NewGuid().ToString(),
						MaxPlayers = 20,
					}.WithDistributedAuthorityNetwork();

					await MultiplayerService.Instance.CreateSessionAsync(options);

					break;
```

**File:** Assets/Anaglyph/LaserTag/Systems.prefab (L282-285)

```text
  ConnectionData:
    Address: 127.0.0.1
    Port: 7777
    ServerListenAddress: 0.0.0.0
```

**File:** Assets/Anaglyph/LaserTag/Interface/Menu/Settings.prefab (L474-477)

```text
      value: By default, headsets try to connect over the local network, but enterprise
        networks can cause issues. Routing connections through a relay server can
        this.
      objectReference: {fileID: 0}
```

**File:** Assets/Anaglyph/XRTemplate/SharedSpaces/MetaAnchors/AutomaticNetworkConnector.cs (L31-33)

```csharp
			OVRColocationSession.ColocationSessionDiscovered += HandleColocationSessionDiscovered;

			HandleChange();
```

**File:** Assets/Anaglyph/XRTemplate/SharedSpaces/MetaAnchors/AutomaticNetworkConnector.cs (L95-112)

```csharp
			switch (transport.Protocol)
			{
				case UnityTransport.ProtocolType.UnityTransport:

					string address = transport.ConnectionData.Address;
					message = LanPrefix + address;
					break;

				case UnityTransport.ProtocolType.RelayUnityTransport:

					var sessionIds = await MultiplayerService.Instance.GetJoinedSessionIdsAsync();
					message = RelayPrefix + sessionIds[sessionIds.Count - 1];

					break;
			}

			Log($"Starting advertisement {message}");
			await OVRColocationSession.StartAdvertisementAsync(Encoding.ASCII.GetBytes(message));
```

**File:** Assets/Anaglyph/LaserTag/Interface/Menu/MultiplayerMenu.cs (L162-165)

```csharp
			var service = useUnityRelayService.Value ? NetworkHelper.Protocol.UnityService : NetworkHelper.Protocol.LAN;

			NetworkHelper.Host(service);
		}
```