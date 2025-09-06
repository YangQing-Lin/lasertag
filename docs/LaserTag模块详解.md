### LaserTag 模块详解（核心玩法）

本文聚焦 `Assets/Anaglyph/LaserTag` 的实现，覆盖玩家与队伍、武器与子弹、场景目标（基地与据点）、对局裁判（MatchReferee/ScoreCounter）以及共位管理的关键逻辑。文中附带代码引用，采用如下格式：`<路径>` (行 a-b) + C# 代码块，便于快速定位。

---

## 概览
- 玩家：`MainPlayer`（本地）、`Networking.Avatar`（网络体）、队伍归属（`TeamOwner`/`Teams`）
- 武器/子弹：`Blaster`、`Automatic`、`Wand`、`Bullet`、`BulletVisuals`、`WeaponsManagement`
- 场景目标：`Base`（基地）、`ControlPoint`（据点/占点）
- 对局：`MatchReferee`（裁判/状态机）、`ScoreCounter`（记分板）
- 共位：`ColocationManager`（按配置选择 Meta Anchor/AprilTag 共位）

---

## 玩家与队伍

### 本地玩家：生成网络体与状态驱动

`Assets/Anaglyph/LaserTag/Player/MainPlayer.cs` (行 89-100)
```csharp
private void SpawnAvatar()
{
    var manager = NetworkManager.Singleton;
    if (!manager.IsConnectedClient)
        return;

    var avatarObject = NetworkObject.InstantiateAndSpawn(avatarPrefab, 
        manager, manager.LocalClientId, destroyWithScene: true, isPlayerObject: true);

    avatar = avatarObject.GetComponent<Networking.Avatar>();
    avatar.TeamOwner.OnTeamChange.AddListener(TeamChanged.Invoke);
}
```

`Assets/Anaglyph/LaserTag/Player/MainPlayer.cs` (行 205-210)
```csharp
if (avatar != null)
{
    avatar.HeadTransform.SetWorldPose(headTransform.GetWorldPose());
    avatar.LeftHandTransform.SetWorldPose(leftHandTransform.GetWorldPose());
    avatar.RightHandTransform.SetWorldPose(rightHandTransform.GetWorldPose());
}
```

`Assets/Anaglyph/LaserTag/Player/MainPlayer.cs` (行 117-125)
```csharp
public void Kill(ulong killedBy)
{
    if (!IsAlive) return;

    WeaponsManagement.canFire = false;
    avatar.isAliveSync.Value = false;
    IsAlive = false;
    Health = 0;
    RespawnTimerSeconds = MatchReferee.Instance.Settings.respawnSeconds;
    avatar.KilledByPlayerRpc(killedBy);
    Died.Invoke();
}
```

### 网络体：生命/得分/被击杀事件

`Assets/Anaglyph/LaserTag/Player/Avatar.cs` (行 121-129)
```csharp
[Rpc(SendTo.Everyone)]
public void DamageRpc(float damage, ulong damagedBy)
{
    if(IsOwner)
        MainPlayer.Instance.Damage(damage, damagedBy);

    onDamaged.Invoke();
}
```

`Assets/Anaglyph/LaserTag/Player/Avatar.cs` (行 130-141)
```csharp
[Rpc(SendTo.Everyone)]
public void KilledByPlayerRpc(ulong killerId) {
    if(AllPlayers.TryGetValue(killerId, out Avatar killer))
        OnPlayerKilledPlayer.Invoke(killer, this);

    var referee = MatchReferee.Instance;
    if(referee.State == MatchState.Playing && killer.Team != Team)
    {
        referee.ScoreTeamRpc(killer.Team, referee.Settings.pointsPerKill);
    }
}
```

### 队伍：同步与常量

`Assets/Anaglyph/LaserTag/Player/Teams/TeamOwner.cs` (行 6-13)
```csharp
public class TeamOwner : NetworkBehaviour
{
    public NetworkVariable<byte> teamSync;
    public byte Team => teamSync.Value;
    public UnityEvent<byte> OnTeamChange = new();
}
```

`Assets/Anaglyph/LaserTag/Player/Teams/Teams.cs` (行 8-15)
```csharp
public static class Teams
{
    public const byte NumTeams = 3;
    public static readonly Color[] Colors = new Color[]
    {
        Color.white,
        new Color(255 / 255f, 25  / 255f, 0   / 255f),
        new Color(0   / 255f, 140 / 255f, 255 / 255f),
    };
}
```

`Assets/Anaglyph/LaserTag/Geo.cs` (行 7-22)
```csharp
public static bool PointIsInCylinder(Vector3 cylPos, float radius, float height, Vector3 point)
{
    if (point.y > cylPos.y + height || point.y < cylPos.y)
        return false;
    cylPos.y = 0;
    point.y = 0;
    float distFlat = Vector3.Distance(cylPos, point);
    if (distFlat > radius)
        return false;
    return true;
}
```

---

## 武器与子弹

### 武器发射与权限

`Assets/Anaglyph/LaserTag/Weapons/WeaponsManagement.cs` (行 1-4)
```csharp
namespace Anaglyph.Lasertag.Weapons {
    public static class WeaponsManagement {
        public static bool canFire = true;
    }
}
```

`Assets/Anaglyph/LaserTag/Weapons/Blaster/Blaster.cs` (行 21-31)
```csharp
public void Fire()
{
    if (!NetworkManager.Singleton.IsConnectedClient || !WeaponsManagement.canFire)
        return;

    NetworkObject n = NetworkObjectPool.Instance.GetNetworkObject(
        boltPrefab, emitFromTransform.position, emitFromTransform.rotation);

    n.SpawnWithOwnership(NetworkManager.Singleton.LocalClientId);
    onFire.Invoke();
}
```

`Assets/Anaglyph/LaserTag/Weapons/Automatic/Automatic.cs` (行 45-56)
```csharp
public void Fire()
{
    if (!NetworkManager.Singleton.IsConnectedClient || !WeaponsManagement.canFire)
        return;

    NetworkObject n = NetworkObjectPool.Instance.GetNetworkObject(
        boltPrefab, emitFromTransform.position, emitFromTransform.rotation);

    n.SpawnWithOwnership(NetworkManager.Singleton.LocalClientId);
    onFire.Invoke();
}
```

`Assets/Anaglyph/LaserTag/Weapons/Wand/Wand.cs` (行 105-118)
```csharp
public void Fire(Pose firePose)
{
    if (!NetworkManager.Singleton.IsConnectedClient || !WeaponsManagement.canFire)
        return;

    trackedPoseDriver.transform.localPosition = firePose.position;
    trackedPoseDriver.transform.localRotation = firePose.rotation;

    NetworkObject n = NetworkObjectPool.Instance.GetNetworkObject(
        boltPrefab, emitFromTransform.position, emitFromTransform.rotation);

    n.SpawnWithOwnership(NetworkManager.Singleton.LocalClientId);
}
```

### 子弹：环境命中 + 物理命中 + 广播

`Assets/Anaglyph/LaserTag/Weapons/Bullets/Bullet.cs` (行 58-64)
```csharp
fireRay = new(transform.position, transform.forward);
if (EnvironmentMapper.Raycast(fireRay, MaxTravelDist, out var envCast))
    if (IsOwner)
        envHitDist = envCast.distance;
    else
        EnvironmentRaycastRpc(envCast.distance);
```

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
        var col = physHit.collider;
        if (col.CompareTag(Networking.Avatar.Tag))
        {
            var av = col.GetComponentInParent<Networking.Avatar>();
            float damage = damageOverDistance.Evaluate(travelDist);
            av.DamageRpc(damage, OwnerClientId);
        }
    }
    else if (didHitEnv)
    {
        Vector3 envHitPoint = fireRay.GetPoint(envHitDist);
        HitRpc(envHitPoint, -transform.forward);
    }
}
```

`Assets/Anaglyph/LaserTag/Weapons/Bullets/Bullet.cs` (行 123-141)
```csharp
[Rpc(SendTo.Everyone)]
private void HitRpc(Vector3 pos, Vector3 norm)
{
    transform.position = pos;
    transform.up = norm;
    isAlive = false;
    OnCollide.Invoke();
    AudioSource.PlayClipAtPoint(collideSFX, transform.position);
    if (IsOwner)
    {
        StartCoroutine(D());
        IEnumerator D() {
            yield return new WaitForSeconds(despawnDelay);
            NetworkObject.Despawn();
        }
    }
}
```

`Assets/Anaglyph/LaserTag/Weapons/Bullets/BulletVisuals.cs` (行 38-53)
```csharp
var manager = NetworkManager.Singleton;
var playerObject = manager.ConnectedClients[bullet.OwnerClientId].PlayerObject;
var teamOwner = playerObject.GetComponent<Networking.Avatar>().TeamOwner;
var team = teamOwner.Team;
var color = Teams.Colors[team];
if (team == 0)
    color = defaultColor;
depthLight.color = color;
propertyBlock.SetColor(TeamColorer.ColorID, color);
trailRenderer.SetPropertyBlock(propertyBlock);
impactEffect.SetVector4(TeamColorer.ColorID, color);
```

---

## 场景目标：基地与据点

### 基地：全局注册与队伍归属

`Assets/Anaglyph/LaserTag/Objects/Gameplay/Base.cs` (行 17-22)
```csharp
public static List<Base> AllBases { get; private set; } = new ();
public UnityEvent<byte> OnTeamChange => teamOwner.OnTeamChange;
```

`Assets/Anaglyph/LaserTag/Objects/Gameplay/Base.cs` (行 34-43)
```csharp
private void Awake()
{
    AllBases.Add(this);
}
public override void OnDestroy()
{
    base.OnDestroy();
    AllBases.Remove(this);
}
```

### 据点：占领进度、控点加分

`Assets/Anaglyph/LaserTag/Objects/Gameplay/ControlPoint.cs` (行 90-103)
```csharp
private async Task ScoreLoop()
{
    if (!IsOwner)
        return;
    while (referee.State == MatchState.Playing)
    {
        await Awaitable.WaitForSecondsAsync(1, destroyCancellationToken);
        destroyCancellationToken.ThrowIfCancellationRequested();
        if (teamOwner.Team != 0)
            referee.ScoreTeamRpc(teamOwner.Team, referee.Settings.pointsPerSecondHoldingPoint);
    }
}
```

`Assets/Anaglyph/LaserTag/Objects/Gameplay/ControlPoint.cs` (行 117-161)
```csharp
if (IsOwner)
{
    bool isSecure = CapturingTeam == HoldingTeam;
    bool capturingTeamIsInside = false;
    bool isStalemated = false;
    if (isSecure)
    {
        foreach (Networking.Avatar player in Networking.Avatar.AllPlayers.Values)
        {
            if (player.Team != 0 && player.Team != HoldingTeam && CheckIfPlayerIsInside(player))
            {
                capturingTeamSync.Value = player.Team;
                isSecure = false;
            }
        }
    }
    else
    {
        foreach (Networking.Avatar player in Networking.Avatar.AllPlayers.Values)
        {
            if (CheckIfPlayerIsInside(player))
            {
                if (player.Team == CapturingTeam)
                    capturingTeamIsInside = true;
                else
                    isStalemated = true;
            }
        }
        if (capturingTeamIsInside)
        {
            if (!isStalemated)
            {
                millisCapturedSync.Value += Time.fixedDeltaTime * 1000;
                if (millisCapturedSync.Value > MillisToTake)
                    CaptureOwnerRpc(CapturingTeam);
            }
        }
        else
        {
            millisCapturedSync.Value = Mathf.Max(0, millisCapturedSync.Value - Time.fixedDeltaTime * 1000);
        }
    }
}
```

---

## 对局：裁判与记分

### 裁判：状态机与胜利条件

`Assets/Anaglyph/LaserTag/Gamemodes/MatchReferee.cs` (行 214-233)
```csharp
private async Task Muster(CancellationToken ctn)
{
    try
    {
        stateSync.Value = MatchState.Mustering;
        while (State == MatchState.Mustering)
        {
            int numPlayersInbase = 0;
            foreach (Networking.Avatar player in Networking.Avatar.AllPlayers.Values)
            {
                if (player.IsInBase)
                    numPlayersInbase++;
            }
            if (numPlayersInbase != 0 && numPlayersInbase == Networking.Avatar.AllPlayers.Count)
            {
                Countdown(ctn);
                break;
            }
            await Awaitable.NextFrameAsync(ctn);
        }
    } catch(OperationCanceledException)
    {
        stateSync.Value = MatchState.NotPlaying;
        throw;
    }
}
```

`Assets/Anaglyph/LaserTag/Gamemodes/MatchReferee.cs` (行 262-301)
```csharp
private async Task RunMatch(MatchSettings settings, CancellationToken ctn)
{
    try
    {
        bool startingNewMatch = stateSync.Value != MatchState.Playing;
        if (startingNewMatch)
        {
            settingsSync.Value = settings;
            stateSync.Value = MatchState.Playing;
            ResetScoresRpc();
        }
        Task timerTask = Task.CompletedTask;
        Task scoreTask = Task.CompletedTask;
        if (settings.CheckWinByTimer())
        {
            float secondsLeft = localTimeMatchEnds - Time.time;
            timerTask = Task.Delay(1000 * Mathf.RoundToInt(secondsLeft), ctn);
        }
        if (settings.CheckWinByPoints())
        {
            winByScoreCompletion = new TaskCompletionSource<bool>();
            ctn.Register(() => winByScoreCompletion.TrySetCanceled(ctn));
            scoreTask = winByScoreCompletion.Task;
        }
        await Task.WhenAll(timerTask, scoreTask);
    }
    catch (OperationCanceledException) { }
    stateSync.Value = MatchState.NotPlaying;
    MatchFinishedRpc();
    ctn.ThrowIfCancellationRequested();
}
```

`Assets/Anaglyph/LaserTag/Gamemodes/MatchReferee.cs` (行 303-320)
```csharp
[Rpc(SendTo.Owner)]
public void ScoreTeamRpc(byte team, int points)
{
    if (State != MatchState.Playing) return;
    if (team == 0 || points == 0) return;
    teamScoresSync[team].Value += points;
    TeamScored.Invoke(team);
    bool canWinByScore = Settings.CheckWinByPoints();
    bool isPlaying = State == MatchState.Playing;
    bool isWinningScore = GetTeamScore(team) > Settings.scoreTarget;
    if (isPlaying && canWinByScore && isWinningScore)
    {
        winByScoreCompletion.SetResult(true);
    }
}
```

### 记分板：达分触发胜利

`Assets/Anaglyph/LaserTag/Gamemodes/ScoreCounter.cs` (行 16-29)
```csharp
for (byte i = 0; i < scores.Length; i++)
{
    scores[i] = new NetworkVariable<int>(0);
    int iCap = i;
    scores[i].OnValueChanged += delegate (int prevSore, int newScore)
    {
        if (scoreTarget > 0 && winningTeam == 0 && prevSore < scoreTarget && newScore >= scoreTarget)
        {
            winningTeam = i;
            OnTeamWin.Invoke(winningTeam);
        }
    };
}
```

---

## 共位管理（LaserTag 入口）

`Assets/Anaglyph/LaserTag/ColocationManager.cs` (行 38-66)
```csharp
public override async void OnNetworkSpawn()
{
    await Awaitable.EndOfFrameAsync();
    EnvironmentMapper.Instance.Clear();
    if (IsOwner)
    {
        colocationMethodSync.Value = useAprilTagColocation.Value ?
            ColocationMethod.AprilTag : ColocationMethod.MetaSharedAnchor;
        aprilTagSizeSync.Value = aprilTagSizeOption.Value / 100f;
    }
    switch (colocationMethodSync.Value)
    {
        case ColocationMethod.MetaSharedAnchor:
            Colocation.SetActiveColocator(metaAnchorColocator);
            break;
        case ColocationMethod.AprilTag:
            Colocation.SetActiveColocator(aprilTagColocator);
            aprilTagColocator.tagSize = aprilTagSizeSync.Value;
            break;
    }
    Colocation.ActiveColocator.Colocate();
}
```

---

## 小结
- 本地玩家 `MainPlayer` 生成并驱动 `Avatar` 网络体；死亡/复活与能否开火统一控制。
- 武器经 `NetworkObjectPool` 池化生成弹丸，子弹 Owner 侧进行环境/物理命中判定并以 RPC 广播结果。
- 据点按圈内人数推进/回退占领，控点队伍在对局中每秒加分。
- `MatchReferee` 作为裁判状态机，统一管理 Mustering/Countdown/Playing 与胜负条件；`ScoreCounter` 作为简单达分判定。
- 共位入口在 `LaserTag/ColocationManager`，根据配置选择 Meta Anchor 或 AprilTag，确保多人空间一致性。


