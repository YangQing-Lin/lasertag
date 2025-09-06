### DriftCorrection 模块详解（漂移校正原型）

本文聚焦 `Assets/Anaglyph/DriftCorrection`，介绍基于 KD 树的最近邻搜索与 ICP（Iterative Closest Point）拟合流程，以及一个场景组件示例如何把“深度点云”对齐至环境模型。此模块偏原型/研究性质。

---

## PointTree（KD 树构建与 KNN 查询）

### 快速排序 + 递归划分构建 KD 树

`Assets/Anaglyph/DriftCorrection/PointTree.cs` (行 71-105)
```csharp
private int BuildBranch(int start, int end, int depth, ref int numNodes)
{
    if (start > end)
        return -1;
    Node node = new();
    node.lesser = -1;
    node.greater = -1;
    int index = numNodes;
    numNodes++;
    int axis = depth % 3;
    QuicksortPoints(points, start, end, axis);
    int split = (start + end) / 2;
    node.point = points[split];
    int length = end - start + 1;
    if(length > 1)
    {
        int nextDepth = depth + 1;
        if(split - 1 >= start)
            node.lesser = BuildBranch(start, split - 1, nextDepth, ref numNodes);
        if(split + 1 <= end)
            node.greater = BuildBranch(split + 1, end, nextDepth, ref numNodes);
    }
    nodes[index] = node;
    return index;
}
```

### KD 树最近邻搜索（并行 JobFor）

`Assets/Anaglyph/DriftCorrection/PointTree.cs` (行 120-132)
```csharp
public void Execute(int index)
{
    int closest = -1;
    float maxDistSqr = float.MaxValue;
    int iterations = 0;
    float3 point = points[index];
    float3 target = math.transform(pointTransform, point);
    Search(target, 0, 0, ref maxDistSqr, ref closest, ref iterations);
    found[index] = nodes[closest].point;
}
```

`Assets/Anaglyph/DriftCorrection/PointTree.cs` (行 134-161)
```csharp
void Search(float3 target, int iNode, int depth, ref float maxDistSqr, ref int closest, ref int iterations)
{
    if (iNode == -1)
        return;
    Node node = nodes[iNode];
    int axis = depth % 3;
    bool targetGreater = target[axis] >= node.point[axis];
    int iSideOfTarget = targetGreater ? node.greater : node.lesser;
    int iOtherSide = targetGreater ? node.lesser : node.greater;
    Search(target, iSideOfTarget, depth + 1, ref maxDistSqr, ref closest, ref iterations);
    float pointDistSqr = math.distancesq(target, node.point);
    if (pointDistSqr <= maxDistSqr)
    {
        maxDistSqr = pointDistSqr;
        closest = iNode;
    }
    float diff = node.point[axis] - target[axis];
    bool splitWithinDistSqr = diff * diff <= maxDistSqr;
    if (splitWithinDistSqr)
        Search(target, iOtherSide, depth + 1, ref maxDistSqr, ref closest, ref iterations);
    iterations++;
}
```

---

## IterativeClosestPoint（ICP 拟合）

### 流程：KNN → 协方差 SVD → 旋转和平移

`Assets/Anaglyph/DriftCorrection/IterativeClosestPoint.cs` (行 10-28)
```csharp
public static async Task<float4x4> Iterate(NativeArray<float3> subject, float4x4 subjTrans, NativeArray<PointTree.Node> targetTree, NativeArray<float3> knnResults)
{
    if(subject.Length != knnResults.Length)
        throw new ArgumentException("KNN results must be same length as target!");
    PointTree.FindClosestPoints findJob = new()
    {
        pointTransform = subjTrans,
        points = subject,
        nodes = targetTree,
        found = knnResults,
    };
    var findJobHandle = findJob.ScheduleParallelByRef(knnResults.Length, 0, default);
    while (!findJobHandle.IsCompleted)
        await Awaitable.NextFrameAsync();
    findJobHandle.Complete();
    return FitCorresponding(subject, subjTrans, knnResults, float4x4.identity);
}
```

`Assets/Anaglyph/DriftCorrection/IterativeClosestPoint.cs` (行 31-63)
```csharp
public static float4x4 FitCorresponding(ReadOnlySpan<float3> subject, float4x4 subjTrans, ReadOnlySpan<float3> target, float4x4 targTrans)
{
    if (subject.Length != target.Length)
        throw new ArgumentException("subject must be same length as target!");
    float3 subjCentroid = float3.zero;
    foreach (float3 v in subject)
        subjCentroid += v;
    subjCentroid /= subject.Length;
    subjCentroid = math.transform(subjTrans, subjCentroid);
    float3 targCentroid = float3.zero;
    foreach (float3 v in target)
        targCentroid += v;
    targCentroid /= target.Length;
    targCentroid = math.transform(targTrans, targCentroid);
    float3x3 covariance = new();
    for (int i = 0; i < target.Length; i++)
    {
        float3 subjPoint = math.transform(subjTrans, subject[i]) - subjCentroid;
        float3 targPoint = math.transform(targTrans, target[i]) - targCentroid;
        for (int x = 0; x < 3; x++)
            for (int y = 0; y < 3; y++)
                covariance[x][y] += targPoint[y] * subjPoint[x];
    }
    quaternion rot = svd.svdRotation(covariance);
    float3 pos = targCentroid - math.mul(rot, subjCentroid);
    float4x4 deltaTrans = float4x4.TRS(pos, rot, new float3(1));
    return deltaTrans;
}
```

---

## EnvironmentDriftCorrection（场景组件示例）

用途：
- 从 `environmentObject` 收集环境网格顶点，构建 KD 树；
- 将 `testDepthMesh`（模拟/实时深度点云）与环境树做 ICP 拟合，输出新的位姿以校正漂移。

`Assets/Anaglyph/DriftCorrection/EnvironmentDriftCorrection.cs` (行 19-27, 35-39)
```csharp
List<float3> vertices = new();
MeshFilter[] meshFilters = environmentObject.GetComponentsInChildren<MeshFilter>();
foreach (var meshFilter in meshFilters)
    foreach (var vertex in meshFilter.mesh.vertices)
        vertices.Add(vertex);
tree = new NativeArray<PointTree.Node>(vertices.Count, Allocator.Persistent);
NativeArray<float3> points = new NativeArray<float3>(vertices.ToArray(), Allocator.TempJob);
// buildJob.Run();
buildJob.Schedule().Complete();
points.Dispose();
```

`Assets/Anaglyph/DriftCorrection/EnvironmentDriftCorrection.cs` (行 51-58)
```csharp
while (enabled)
{
    float4x4 trans = transform.localToWorldMatrix;
    Matrix4x4 mat = await IterativeClosestPoint.Iterate(newPoints, trans, tree, knnResults);
    mat = mat * transform.localToWorldMatrix;
    testDepthMesh.transform.SetPositionAndRotation(mat.GetPosition(), mat.rotation);
}
```

---

## 小结与建议
- 该模块演示了用 KD 树 + ICP 做点云对齐，流程清晰：构建树 → 最近邻 → 质心/协方差 → SVD 求旋转 → 组装 TRS。
- 当前示例未做鲁棒性与收敛性优化（如 outlier 剔除、加权、迭代终止条件、多次迭代累计更新等），如果用于生产建议：
  - 增加迭代阈值和最大迭代次数；
  - 对异常配对点进行剔除或加权；
  - 将 `mat` 累乘到累计变换上而非仅应用一次增量；
  - 使用 `Burst`/`Jobs` 扩大并行度并监控分配与回收（NativeArray 生命周期管理）。


