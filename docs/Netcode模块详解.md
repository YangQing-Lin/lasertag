### Netcode 模块详解（联网封装与对象池）

本文聚焦 `Assets/Anaglyph/Netcode` 的实现，解析其网络连接流程、会话与状态管理、LAN 连接细节、对象池资源管理机制，并说明"局域网中的地形/环境交互"在本项目中的处理方式（并非网络同步，而是通过环境深度本地推断）。

---

## 目录
- 网络连接与状态（`NetcodeHelper`）
- 可序列化姿态（`NetworkPose`）
- 网络对象池与 Prefab Handler（`NetworkObjectPool`）
- 连接态 UI 辅助（`ActiveIfConnected` / `InteractiveIfConnected`）
- 局域网中的"地形/环境交互"说明（`Bullet` + `EnvironmentMapper`）

---

## 网络连接与状态：`Anaglyph/Netcode/NetcodeHelper.cs`

职责概述：
- 统一封装 Host/Client 在 LAN 与 Unity Services 两种模式下的连接流程。
- 管理全局连接状态（Disconnected/Connecting/Connected），并透出 `StateChange` 事件给 UI/系统使用。
- 在 Unity Services 模式下完成服务初始化、匿名登录、会话创建/加入与冷却控制。

关键点：
- 状态驱动：

`Assets/Anaglyph/Netcode/NetcodeHelper.cs` (行 12-50)
```csharp
public static event Action<NetworkState> StateChange = delegate { };
public static NetworkState State
{
    get => _state;
    private set
    {
        bool changed = value != _state;
        _state = value;
        if (changed)
            StateChange?.Invoke(_state);
    }
}
```

- 加载后挂接连接事件：

`Assets/Anaglyph/Netcode/NetcodeHelper.cs` (行 52-58)
```csharp
[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.AfterSceneLoad)]
private static void OnSceneLoad()
{
    manager.OnClientStarted += () => State = NetworkState.Connecting;
    manager.OnClientStopped += _ => State = NetworkState.Disconnected;
    manager.OnConnectionEvent += OnConnectionEvent;
}
```

- LAN 主机与客户端：

`Assets/Anaglyph/Netcode/NetcodeHelper.cs` (行 94-116)
```csharp
private static async Task Host(Protocol protocol, CancellationToken ct)
{
    switch (protocol)
    {
        case Protocol.LAN:
            SetNetworkTransportType("UnityTransport");
            manager.NetworkConfig.UseCMBService = false;
            // 解析本机 IPv4
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
            manager.StartHost();
            break;
    }
}
```

`Assets/Anaglyph/Netcode/NetcodeHelper.cs` (行 126-136)
```csharp
public static void ConnectLAN(string ip)
{
    State = NetworkState.Connecting;
    SetNetworkTransportType("UnityTransport");
    manager.NetworkConfig.UseCMBService = false;
    transport.SetConnectionData(ip, port);
    manager.StartClient();
}
```

- Unity Services 会话：

`Assets/Anaglyph/Netcode/NetcodeHelper.cs` (行 158-186)
```csharp
private static async Task ConnectUnityServices(string id, CancellationToken ct)
{
    SetNetworkTransportType("DistributedAuthorityTransport");
    manager.NetworkConfig.UseCMBService = true;
    State = NetworkState.Connecting;
    // 冷却窗口
    if (Time.time < cooldownDoneTime) {
        float waitTime = cooldownDoneTime - Time.time;
        await Awaitable.WaitForSecondsAsync(waitTime);
    }
    if (ct.IsCancellationRequested) return;
    cooldownDoneTime = Time.time + cooldownSeconds;
    await SetupServices(); // 初始化 UnityServices + 匿名登录
    if (ct.IsCancellationRequested) return;
    var options = new SessionOptions(){ Name = id, MaxPlayers = 20, }.WithDistributedAuthorityNetwork();
    CurrentSessionName = id;
    CurrentSession = await MultiplayerService.Instance.CreateOrJoinSessionAsync(id, options);
    if (ct.IsCancellationRequested) Disconnect();
}
```

结论：
- LAN 模式下使用 `UnityTransport`，直接以 IPv4 地址和端口连入主机。
- 云模式下走分布式权限传输与 Unity Services，会话由 `SessionOptions` 管理。
- 连接状态机与事件回调集中于 `NetcodeHelper`，UI 可订阅 `StateChange` 进行反馈。

## 网络拓扑与路由/中继要求

- LAN（局域网直连，UnityTransport）
  - 拓扑：主机权威（Host Authoritative）的星型拓扑，`StartHost()` 的设备既是权威也充当转发点，客户端用 `StartClient()` 直连主机。
  - 端口：默认 UDP 7777，可在代码中调整：

`Assets/Anaglyph/Netcode/NetcodeHelper.cs` (行 16-24)
```csharp
public static ushort port = 7777;
...
private static UnityTransport transport => (UnityTransport)NetworkManager.Singleton.NetworkConfig.NetworkTransport;
```

  - 传输配置：LAN 下强制使用 `UnityTransport` 且禁用 CMB 服务：

`Assets/Anaglyph/Netcode/NetcodeHelper.cs` (行 98-106, 114-116, 126-136)
```csharp
SetNetworkTransportType("UnityTransport");
manager.NetworkConfig.UseCMBService = false;
...
transport.SetConnectionData(localAddressOrIp, port);
manager.StartHost(); // 或 StartClient()
```

  - 路由/防火墙：无需额外“专用路由器/中继设备”。同一局域网内即可通信；需确保主机防火墙放行 UDP 7777（或所配置端口）。若要跨公网直连主机，则需要自行进行端口映射/UPnP，项目未内置公网 Relay。

- Unity Services（分布式权限网络，DistributedAuthorityTransport）
  - 拓扑：通过 Unity Services Multiplayer 的分布式权限网络建立覆盖连接，无单一自建专服；服务侧负责 NAT 穿透/中继。
  - 传输与会话：

`Assets/Anaglyph/Netcode/NetcodeHelper.cs` (行 158-163, 178-186)
```csharp
SetNetworkTransportType("DistributedAuthorityTransport");
manager.NetworkConfig.UseCMBService = true;
...
var options = new SessionOptions(){ Name = id, MaxPlayers = 20, }
    .WithDistributedAuthorityNetwork();
CurrentSession = await MultiplayerService.Instance.CreateOrJoinSessionAsync(id, options);
```

  - 路由需求：不需要自备路由/中继或端口映射，只要设备能够访问互联网即可（企业/校园网络需允许访问 Unity 服务域名与相关端口）。

---

## 可序列化姿态：`Anaglyph/Netcode/NetworkPose.cs`

用途：通过 Netcode 传输位置与旋转（如弹丸初始姿态）。

关键代码：

`Assets/Anaglyph/Netcode/NetworkPose.cs` (行 6-20)
```csharp
public struct NetworkPose : INetworkSerializable, System.IEquatable<NetworkPose>
{
    public Vector3 position;
    public Quaternion rotation;
    public NetworkPose(Transform transform)
    {
        this.position = transform.position;
        this.rotation = transform.rotation;
    }
}
```

`Assets/Anaglyph/Netcode/NetworkPose.cs` (行 42-56)
```csharp
public void NetworkSerialize<T>(BufferSerializer<T> serializer) where T : IReaderWriter
{
    if (serializer.IsReader)
    {
        var reader = serializer.GetFastBufferReader();
        reader.ReadValueSafe(out position);
        reader.ReadValueSafe(out rotation);
    }
    else
    {
        var writer = serializer.GetFastBufferWriter();
        writer.WriteValueSafe(position);
        writer.WriteValueSafe(rotation);
    }
}
```

---

## 网络对象池与 Prefab Handler：`Anaglyph/Netcode/NetworkObjectPool.cs`

职责概述：
- 在客户端启动时注册需要池化的网络 Prefab，按配置预热若干实例。
- 拦截 Netcode 的 Spawn/Destroy，改为从池获取/归还，减少运行时 GC 与 Instantiate/Destroy 成本。

关键注册与生命周期：

`Assets/Anaglyph/Netcode/NetworkObjectPool.cs` (行 41-48)
```csharp
private void OnSessionStart()
{
    // Registers all objects in PooledPrefabsList to the cache.
    foreach (var configObject in PooledPrefabsList)
    {
        RegisterPrefabInternal(configObject.Prefab, configObject.PrewarmCount);
    }
}
```

`Assets/Anaglyph/Netcode/NetworkObjectPool.cs` (行 146-149)
```csharp
// Register Netcode Spawn handlers
NetworkManager.Singleton.PrefabHandler.AddHandler(prefab, new PooledPrefabInstanceHandler(prefab, this));
```

取用与归还：

`Assets/Anaglyph/Netcode/NetworkObjectPool.cs` (行 88-96)
```csharp
public NetworkObject GetNetworkObject(GameObject prefab, Vector3 position, Quaternion rotation)
{
    var networkObject = m_PooledObjects[prefab].Get();
    networkObject.transform.SetPositionAndRotation(position, rotation);
    return networkObject;
}
```

`Assets/Anaglyph/Netcode/NetworkObjectPool.cs` (行 170-178)
```csharp
NetworkObject INetworkPrefabInstanceHandler.Instantiate(ulong ownerClientId, Vector3 position, Quaternion rotation)
{
    return m_Pool.GetNetworkObject(m_Prefab, position, rotation);
}
void INetworkPrefabInstanceHandler.Destroy(NetworkObject networkObject)
{
    m_Pool.ReturnNetworkObject(networkObject, m_Prefab);
}
```

结论：
- 该池在"客户端开始"事件时集中注册并预热，随后作为 Netcode 的 Prefab 实例/销毁处理器。
- 武器发射时通过 `NetworkObjectPool.Instance.GetNetworkObject(...).SpawnWithOwnership(...)` 即可走池化路线。

---

## 连接态 UI 辅助

- `ActiveIfConnected.cs`：根据连接事件显隐对象。

`Assets/Anaglyph/Netcode/ActiveIfConnected.cs` (行 8-13)
```csharp
NetworkManager.Singleton.OnConnectionEvent += OnConnectionEvent;
gameObject.SetActive(NetworkManager.Singleton.IsConnectedClient);
```

- `InteractiveIfConnected.cs`：根据是否连接来设置 `Selectable.interactable`。

`Assets/Anaglyph/Netcode/InteractiveIfConnected.cs` (行 42-46)
```csharp
bool isConnected = networkManager != null && networkManager.IsConnectedClient;
selectable.interactable = isConnected ^ invert;
```

---

## 局域网中的"地形/环境交互"说明

问题澄清：本项目没有"地形网格的网络同步"。环境命中/遮挡等效果通过 XR 环境深度在本地完成，典型用例是子弹运动与环境命中判定：

- `LaserTag/Weapons/Bullets/Bullet.cs` 中，Owner 侧在 Spawn 后计算环境命中距离：

`Assets/Anaglyph/LaserTag/Weapons/Bullets/Bullet.cs` (行 58-64)
```csharp
fireRay = new(transform.position, transform.forward);
if (EnvironmentMapper.Raycast(fireRay, MaxTravelDist, out var envCast))
    if (IsOwner)
        envHitDist = envCast.distance;
    else
        EnvironmentRaycastRpc(envCast.distance);
```

- 逐帧推进并对比环境命中：

`Assets/Anaglyph/LaserTag/Weapons/Bullets/Bullet.cs` (行 90-118)
```csharp
if (IsOwner)
{
    bool didHitEnv = travelDist > envHitDist;
    if (didHitEnv)
        transform.position = fireRay.GetPoint(envHitDist);
    bool didHitPhys = Physics.Linecast(prevPos, transform.position, out var physHit,
        Physics.DefaultRaycastLayers, QueryTriggerInteraction.Ignore);
    if (didHitPhys)
    {
        HitRpc(physHit.point, physHit.normal);
        // 命中玩家则 RPC 伤害
    }
    else if (didHitEnv)
    {
        Vector3 envHitPoint = fireRay.GetPoint(envHitDist);
        HitRpc(envHitPoint, -transform.forward);
    }
}
```

对应环境深度体素化与射线：

`Assets/Anaglyph/XRTemplate/Depth/EnvironmentMapper.cs` (行 198-206)
```csharp
public static bool Raycast(Ray ray, float maxDist, out RayResult result, bool fallback = false)
    => Instance.RaycastInternal(ray, maxDist, out result, fallback);
```

因此，"地形/环境交互"是客户端基于本机环境深度作出的本地判定，并非在 LAN 上同步传输地形网格或体素数据。命中结果通过 RPC（如 `HitRpc`）广播，以统一表现。

---

## 小结
- 连接：`NetcodeHelper` 统一封装 LAN 与 Unity Services，管理状态与冷却；LAN 使用 `UnityTransport` 与 IP:Port 直连；云端使用 `DistributedAuthorityTransport` + Unity Services 分布式权限会话。
- 资源：`NetworkObjectPool` 作为 Netcode 的 Prefab Handler，完成网络对象的池化获取与归还，显著降低运行时开销。
- 地形/环境：不做网络同步；通过本地 `EnvironmentMapper` 的深度体素射线完成命中推断，命中事件用 RPC 同步表现。
- 路由与中继：两种模式均无需额外“专用路由器/中继设备”。LAN 需同网段并放行 UDP 7777；云端由 Unity 服务侧处理 NAT 穿透/中继（仅要求可访问互联网）。
