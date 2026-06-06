# UniversalCable "无存储纯传递"架构可行性分析

## 一、当前架构概述

### 1.1 能量存储层级

Mekanism 的能量传输采用**双层缓冲架构**：

```
┌─────────────────────────────────────────┐
│         EnergyNetwork (网络层)           │
│  - VariableCapacityEnergyContainer      │
│    存储整个网络的总能量                   │
│  - currentScale: 客户端渲染用填充比例     │
└──────────────┬──────────────────────────┘
               │
               │ absorbBuffer() / releaseShare()
               │
┌──────────────▼──────────────────────────┐
│      UniversalCable (线缆层)             │
│  - BasicEnergyContainer buffer          │
│    每个线缆的本地缓冲                     │
│  - FloatingLong lastWrite               │
│    NBT 持久化用的最后写入值              │
└─────────────────────────────────────────┘
```

### 1.2 关键代码路径追踪

#### 网络合并时的能量处理
**文件**: `EnergyNetwork.java:58-71`
```java
public List<UniversalCable> adoptTransmittersAndAcceptorsFrom(EnergyNetwork net) {
    // 1. 计算两个网络的能量比例
    FloatingLong ourScale = currentScale == 0 ? FloatingLong.ZERO : oldCapacity.multiply(currentScale);
    FloatingLong theirScale = net.currentScale == 0 ? FloatingLong.ZERO : net.getCapacityAsFloatingLong().multiply(net.currentScale);

    // 2. 合并能量到当前网络容器
    if (!isRemote() && !net.energyContainer.isEmpty()) {
        energyContainer.setEnergy(energyContainer.getEnergy().add(net.getBuffer()));
        net.energyContainer.setEmpty();
    }
    return transmittersToUpdate;
}
```

#### 线缆加入网络时的能量吸收
**文件**: `DynamicBufferedNetwork.java:57-63` + `EnergyNetwork.java:80-85`
```java
// DynamicBufferedNetwork.addTransmitterFromCommit()
protected void addTransmitterFromCommit(TRANSMITTER transmitter) {
    super.addTransmitterFromCommit(transmitter);
    chunks.add(ChunkPos.asLong(transmitter.getTilePos()));
    updateCapacity(transmitter);
    absorbBuffer(transmitter);  // ← 调用子类实现
}

// EnergyNetwork.absorbBuffer()
public void absorbBuffer(UniversalCable transmitter) {
    FloatingLong energy = transmitter.releaseShare();  // 清空线缆 buffer
    if (!energy.isZero()) {
        energyContainer.setEnergy(energyContainer.getEnergy().add(energy));  // 累加到网络
    }
}
```

#### 网络失效时的能量返还
**文件**: `DynamicNetwork.java:101-119`
```java
public void invalidate(@Nullable TRANSMITTER triggerTransmitter) {
    removeInvalid(triggerTransmitter);
    if (!isRemote()) {
        for (TRANSMITTER transmitter : transmitters) {
            if (transmitter.isValid()) {
                transmitter.takeShare();  // ← 从网络取回能量到本地 buffer
                transmitter.setTransmitterNetwork(null);
                TransmitterNetworkRegistry.registerOrphanTransmitter(transmitter);
            }
        }
    }
    deregister();
}
```

**UniversalCable.takeShare()** (`UniversalCable.java:199-211`):
```java
public void takeShare() {
    if (hasTransmitterNetwork()) {
        EnergyNetwork transmitterNetwork = getTransmitterNetwork();
        if (!transmitterNetwork.energyContainer.isEmpty() && !lastWrite.isZero()) {
            // 从网络容器中减去 lastWrite，恢复到本地 buffer
            transmitterNetwork.energyContainer.setEnergy(
                transmitterNetwork.energyContainer.getEnergy().subtract(lastWrite));
            buffer.setEnergy(lastWrite);
        }
    } else {
        // 无网络时，确保 lastWrite 与 buffer 同步
        lastWrite = buffer.getEnergy();
    }
}
```

#### Tick 更新与能量分发
**文件**: `EnergyNetwork.java:151-163`
```java
public void onUpdate() {
    super.onUpdate();
    if (needsUpdate) {
        MinecraftForge.EVENT_BUS.post(new EnergyTransferEvent(this));
        needsUpdate = false;
    }
    if (energyContainer.isEmpty()) {
        prevTransferAmount = FloatingLong.ZERO;
    } else {
        // 从网络容器提取全部能量，尝试分发给所有接受者
        prevTransferAmount = tickEmit(energyContainer.getEnergy());
        energyContainer.extract(prevTransferAmount, Action.EXECUTE, AutomationType.INTERNAL);
    }
}
```

**tickEmit()** (`EnergyNetwork.java:130-143`):
```java
private FloatingLong tickEmit(FloatingLong energyToSend) {
    Collection<Map<Direction, LazyOptional<IStrictEnergyHandler>>> acceptorValues = 
        acceptorCache.getAcceptorValues();
    EnergyAcceptorTarget target = new EnergyAcceptorTarget(acceptorValues.size() * 2);

    // 收集所有能接收能量的接受者
    for (Map<Direction, LazyOptional<IStrictEnergyHandler>> acceptors : acceptorValues) {
        for (LazyOptional<IStrictEnergyHandler> lazyAcceptor : acceptors.values()) {
            lazyAcceptor.ifPresent(acceptor -> {
                if (acceptor.insertEnergy(energyToSend, Action.SIMULATE)
                        .smallerThan(energyToSend)) {
                    target.addHandler(acceptor);
                }
            });
        }
    }
    // 使用 EmitUtils 公平分配算法分发能量
    return EmitUtils.sendToAcceptors(target, energyToSend.copy());
}
```

---

## 二、"无存储纯传递"架构的技术难点

### 2.1 核心挑战：能量守恒与状态管理

#### 难点 1：网络合并时的能量来源问题

**当前逻辑依赖**:
```
网络A (100 FE) + 网络B (200 FE) → 合并后网络 (300 FE)
```

合并时使用 `currentScale`（填充比例）来计算实际能量：
```java
FloatingLong ourScale = oldCapacity.multiply(currentScale);       // 1000 * 0.1 = 100
FloatingLong theirScale = otherCapacity.multiply(otherScale);     // 2000 * 0.1 = 200
energyContainer.setEnergy(ourScale.add(theirScale));              // 100 + 200 = 300
```

**如果移除线缆存储**:
- 线缆没有 `buffer`，无法通过 `releaseShare()` 提供能量
- 网络合并时必须完全依赖 `energyContainer`，但 `currentScale` 的计算需要容量和比例
- **问题**: 如果网络在合并前未执行过 `onUpdate()`，`currentScale` 可能为 0，导致能量计算错误

#### 难点 2：区块加载时的能量恢复时序

**当前流程**:
```
1. TileEntity.load() → UniversalCable.read() → buffer.setEnergy(lastWrite)
2. clearRemoved() → registerOrphanTransmitter()
3. ServerTickEvent → assignOrphans() → 网络合并
4. commit() → absorbBuffer() → 将 buffer 能量转移到网络容器
```

**如果移除线缆存储**:
- 步骤 1 无法恢复能量到 buffer（因为没有 buffer）
- 步骤 4 的 `absorbBuffer()` 将始终返回 ZERO
- **必须修改**: 需要在网络级别直接持久化能量，而不是分散到各个线缆

#### 难点 3：网络失效时的能量返还

**当前逻辑**:
```java
// DynamicNetwork.invalidate()
for (TRANSMITTER transmitter : transmitters) {
    transmitter.takeShare();  // 从网络取回 lastWrite 对应的能量
}
```

**UniversalCable.takeShare()**:
```java
if (!transmitterNetwork.energyContainer.isEmpty() && !lastWrite.isZero()) {
    transmitterNetwork.energyContainer.setEnergy(
        networkEnergy.subtract(lastWrite));
    buffer.setEnergy(lastWrite);
}
```

**如果移除线缆存储**:
- `takeShare()` 无法将能量恢复到 buffer
- 网络失效后，能量会"卡"在网络容器中，但网络已经 deregister
- **风险**: 可能导致能量丢失或无法访问

### 2.2 能力接口（Capability）暴露问题

**TileEntityUniversalCable.getEnergyContainers()** (`TileEntityUniversalCable.java:95-97`):
```java
private List<IEnergyContainer> getEnergyContainers(@Nullable Direction side) {
    return energyHandlerManager.getContainers(side);
}
```

**UniversalCable.getEnergyContainers()** (`UniversalCable.java:89-94`):
```java
public List<IEnergyContainer> getEnergyContainers(@Nullable Direction side) {
    if (hasTransmitterNetwork()) {
        return getTransmitterNetwork().getEnergyContainers(side);  // 返回网络容器
    }
    return energyContainers;  // 返回本地 buffer
}
```

**问题分析**:
- 当线缆属于网络时，外部机器通过 Capability 访问的是**网络级别的容器**
- 当线缆是孤儿时，访问的是**本地 buffer**
- **如果移除本地 buffer**: 孤儿状态的线缆将无法暴露任何能量容器，导致：
  - 发电机无法向孤立线缆注入能量
  - 机器无法从孤立线缆抽取能量
  - 新放置的线缆在未连接网络前完全无用

### 2.3 客户端同步与渲染

**TileEntityUniversalCable.getUpdateTag()** (`TileEntityUniversalCable.java:84-93`):
```java
public CompoundTag getUpdateTag() {
    CompoundTag updateTag = super.getUpdateTag();
    if (getTransmitter().hasTransmitterNetwork()) {
        EnergyNetwork network = getTransmitter().getTransmitterNetwork();
        updateTag.putString(NBTConstants.ENERGY_STORED, network.energyContainer.getEnergy().toString());
        updateTag.putFloat(NBTConstants.SCALE, network.currentScale);
    }
    return updateTag;
}
```

**EnergyNetwork.computeContentScale()** (`EnergyNetwork.java:166-175`):
```java
protected float computeContentScale() {
    float scale = (float) energyContainer.getEnergy().divideToLevel(energyContainer.getMaxEnergy());
    float ret = Math.max(currentScale, scale);
    // 平滑过渡动画
    if (!prevTransferAmount.isZero() && ret < 1) {
        ret = Math.min(1, ret + 0.02F);
    } else if (prevTransferAmount.isZero() && ret > 0) {
        ret = Math.max(scale, ret - 0.02F);
    }
    return ret;
}
```

**如果移除线缆存储**:
- `currentScale` 的计算仍然依赖 `energyContainer.getEnergy()`
- 但 `prevTransferAmount` 的更新依赖每 tick 的分发逻辑
- **影响**: 客户端渲染可能需要调整，因为不再有"本地缓冲"的概念

---

## 三、对现有网络合并逻辑的影响

### 3.1 合并流程对比

#### 当前架构的合并流程
```
网络A (容量1000, 能量100, scale=0.1)
网络B (容量2000, 能量200, scale=0.1)

1. adoptTransmittersAndAcceptorsFrom():
   - ourScale = 1000 * 0.1 = 100
   - theirScale = 2000 * 0.1 = 200
   - 新容量 = 3000
   - 新 scale = (100+200) / 3000 = 0.1
   - energyContainer.setEnergy(100 + 200) = 300

2. commit() → addTransmitterFromCommit() → absorbBuffer():
   - 遍历所有新加入的线缆
   - 每个线缆的 buffer.releaseShare() 清空并累加到网络
   - （实际上此时 buffer 应该已经是空的，因为能量已在步骤1合并）
```

#### 无存储架构的合并流程（假设）
```
网络A (容量1000, 能量100, scale=0.1)
网络B (容量2000, 能量200, scale=0.1)

1. adoptTransmittersAndAcceptorsFrom():
   - 同上，能量合并在网络层完成 ✓

2. commit() → addTransmitterFromCommit() → absorbBuffer():
   - 线缆没有 buffer，releaseShare() 返回 ZERO
   - 无需额外操作 ✓

潜在问题:
- 如果步骤1中 currentScale 未正确初始化（例如为0），能量会丢失
- 孤儿线缆无法暂存能量，必须在加入网络前就有网络容器
```

### 3.2 孤儿传输器处理

**当前机制**:
- 孤儿线缆有本地 `buffer`，可以独立存储能量
- 当多个孤儿通过 BFS 发现彼此时，创建新网络并合并能量
- `EnergyNetwork` 构造函数创建空的 `energyContainer`，然后通过 `adoptAllAndRegister()` 合并

**无存储架构的问题**:
```java
// EnergyNetwork 构造函数
public EnergyNetwork(UUID networkID) {
    super(networkID);
    energyContainer = VariableCapacityEnergyContainer.create(...);  // 初始能量为 0
}

public EnergyNetwork(Collection<EnergyNetwork> networks) {
    this(UUID.randomUUID());
    adoptAllAndRegister(networks);  // 合并现有网络的能量
}
```

**场景**: 3个孤儿线缆各自有 50 FE，合并成一个网络
- **当前**: 每个线缆的 buffer 有 50 FE，合并时通过 `absorbBuffer()` 累加到网络容器 = 150 FE ✓
- **无存储**: 孤儿线缆没有 buffer，能量存储在哪里？
  - **方案A**: 仍然保留最小 buffer（如 1 FE），仅用于临时存储
  - **方案B**: 在创建网络前先收集所有孤儿的能量（需要重构 `OrphanPathFinder`）
  - **方案C**: 禁止孤儿线缆存储能量（功能退化，不推荐）

---

## 四、高频 Tick 下的性能与能量丢失风险

### 4.1 性能分析

#### 当前架构的 Tick 开销
```
每 tick (Server Side):
1. TileEntityUniversalCable.onUpdateServer():
   - pullFromAcceptors(): 从相邻机器拉取能量到网络容器
   - 复杂度: O(连接的接受者数量)

2. TransmitterNetworkRegistry.onTick():
   - handleChangedChunks(): O(变化的区块数)
   - removeInvalidTransmitters(): O(无效传输器数)
   - assignOrphans(): O(孤儿传输器数 × BFS深度)
   - commit(): O(待添加的传输器数)
   - 所有网络.onUpdate(): O(网络数 × 接受者数)

3. EnergyNetwork.onUpdate():
   - tickEmit(): 遍历所有接受者，模拟插入，公平分配
   - 复杂度: O(接受者数量²) 最坏情况
```

#### 无存储架构的性能影响
**正面影响**:
- 移除 `absorbBuffer()` 和 `releaseShare()` 的调用，减少少量开销
- 简化 NBT 读写（无需处理 `lastWrite`）

**负面影响**:
- **无明显性能提升**: 瓶颈在于 `tickEmit()` 的公平分配算法，而非 buffer 管理
- **可能增加开销**: 如果需要在每次外部访问时动态计算能量，反而会增加计算量

### 4.2 能量丢失风险分析

#### 风险场景 1：服务器崩溃

**当前架构**:
```
T0: 网络容器有 1000 FE
T1: 服务器崩溃，未执行 saveAdditional()
重启后:
- 如果区块已保存: 读取 lastWrite（可能是旧值）
- 如果区块未保存: 能量丢失
```

**无存储架构**:
```
T0: 网络容器有 1000 FE
T1: 服务器崩溃
重启后:
- 网络容器的能量需要通过网络 ID 持久化（当前未实现）
- 如果仅依赖线缆的 lastWrite，而线缆没有 buffer，则无法恢复
```

**结论**: 无存储架构**不会改善**崩溃恢复，反而可能恶化，除非实现网络级别的持久化。

#### 风险场景 2：网络分裂与合并

**当前架构**:
```
网络 (1000 FE) → 破坏中间线缆 → 分裂为网络A和网络B
- invalidate() 调用每个线缆的 takeShare()
- 每个线缆根据 lastWrite 从网络取回能量
- 如果 lastWrite 不准确，能量分配不均但不会丢失
```

**无存储架构**:
```
网络 (1000 FE) → 破坏中间线缆 → 分裂
- takeShare() 无法将能量恢复到线缆（无 buffer）
- 网络分裂后，两个新网络如何分配原网络的能量？
- **高风险**: 可能导致能量重复计算或丢失
```

#### 风险场景 3：区块卸载/加载

**当前架构**:
```
区块卸载:
- onChunkUnloaded() → takeShare()
- 线缆从网络取回 lastWrite 对应的能量到 buffer
- 区块数据保存时，buffer 的能量通过 lastWrite 持久化

区块加载:
- load() → buffer.setEnergy(lastWrite)
- 重新加入网络时，absorbBuffer() 将能量转移回网络
```

**无存储架构**:
```
区块卸载:
- takeShare() 无法恢复能量到 buffer
- 区块保存时，线缆没有能量数据可写

区块加载:
- load() 无法恢复能量
- **能量丢失**
```

**结论**: 无存储架构在区块卸载/加载场景下**必然导致能量丢失**，除非实现网络跨区块的持久化。

---

## 五、技术实现的主要难点总结

### 5.1 必须解决的核心问题

| 问题 | 难度 | 影响 |
|------|------|------|
| **孤儿线缆的能量存储** | 高 | 新放置的线缆无法工作 |
| **网络合并时的能量计算** | 中 | 依赖 currentScale，可能因未初始化而出错 |
| **网络分裂时的能量分配** | 高 | 可能导致能量丢失或重复 |
| **区块卸载/加载的能量持久化** | 极高 | 需要重构整个持久化系统 |
| **Capability 接口的兼容性** | 中 | 孤儿状态无法暴露能量容器 |

### 5.2 需要重构的模块

1. **持久化系统** (最高优先级)
   - 当前: 能量分散存储在每个线缆的 `lastWrite`
   - 需要: 网络级别的持久化，通过 `networkID` 关联
   - 涉及文件:
     - `EnergyNetwork.java`: 添加网络数据的保存/加载
     - `MekanismSavedData`: 注册网络数据存储
     - `CommonWorldTickHandler.java`: 在世界加载时恢复网络数据

2. **孤儿传输器处理**
   - 当前: 孤儿线缆通过 `buffer` 暂存能量
   - 需要: 要么保留最小 buffer，要么在创建网络时立即分配能量
   - 涉及文件:
     - `OrphanPathFinder` (TransmitterNetworkRegistry.java)
     - `EnergyNetwork` 构造函数

3. **网络分裂逻辑**
   - 当前: `invalidate()` 通过 `takeShare()` 返还能量
   - 需要: 实现基于容量的公平分配算法
   - 涉及文件:
     - `DynamicNetwork.invalidate()`
     - `UniversalCable.takeShare()`

4. **Capability 暴露**
   - 当前: 孤儿线缆暴露本地 buffer
   - 需要: 即使无 buffer，也要提供虚拟容器接口
   - 涉及文件:
     - `UniversalCable.getEnergyContainers()`
     - `EnergyHandlerManager`

### 5.3 替代方案建议

鉴于"完全无存储"架构的高风险和复杂重构需求，建议考虑以下**折中方案**：

#### 方案 A: 最小化缓冲（推荐）
- 保留 `buffer`，但将其容量设为极小值（如 1 FE）
- 主要能量仍存储在 `EnergyNetwork.energyContainer`
- 优点:
  - 保持现有架构兼容性
  - 解决孤儿线缆的能量存储问题
  - 简化 NBT 持久化（只需保存网络级能量）
- 缺点:
  - 仍有少量缓冲，不是真正的"纯传递"

#### 方案 B: 改进持久化逻辑（次推荐）
- 保持当前双层缓冲架构
- 修复 `lastWrite` 同步问题（已通过 `forceUpdateSaveShares()` 解决）
- 添加网络级别的校验和备份
- 优点:
  - 最小改动，风险低
  - 已部分实施（见之前的修复）
- 缺点:
  - 仍存在缓冲延迟

#### 方案 C: 完全无存储（不推荐）
- 移除所有线缆级别的能量存储
- 实现完整的网络级持久化
- 优点:
  - 理论上消除缓冲延迟
- 缺点:
  - 重构工作量巨大
  - 引入新的能量丢失风险
  - 性能提升不明显

---

## 六、结论与建议

### 6.1 直接回答用户问题

**Q1: 是否能解决能量滞留问题？**
- **答**: 不能根本解决。能量滞留的根本原因是网络分发算法（`tickEmit()`）每 tick 只执行一次，与是否有线缆缓冲无关。即使移除缓冲，能量仍需在网络容器中等待分发。

**Q2: 是否能解决同步延迟问题？**
- **答**: 可能略微改善，但不明显。同步延迟主要来自：
  1. 网络合并时的 BFS 搜索（ unavoidable）
  2. `commit()` 后的客户端同步包发送
  3. 移除缓冲不会加速这些过程

**Q3: 技术实现的主要难点是什么？**
- **答**: 见 5.1 节，核心难点是孤儿线缆能量存储、网络分裂能量分配、区块持久化。

**Q4: 对网络合并逻辑的影响？**
- **答**: 见 3.1 节，主要影响是 `absorbBuffer()` 变为空操作，但需要确保合并前的能量计算正确。

**Q5: 是否会导致高频 tick 下的性能问题或能量丢失？**
- **答**:
  - 性能: 无明显提升，瓶颈不在 buffer 管理
  - 能量丢失: **高风险**，特别是在区块卸载/加载、网络分裂、服务器崩溃场景

### 6.2 最终建议

**不建议实施"完全无存储纯传递"架构**，原因：
1. 重构成本远高于收益
2. 引入新的能量丢失风险
3. 性能提升微乎其微
4. 当前的 NBT 持久化问题已通过 `forceUpdateSaveShares()` 修复

**推荐行动**:
1. 部署已实施的 `forceUpdateSaveShares()` 修复
2. 在 Mohist 环境下测试验证
3. 如果仍有问题，考虑方案 A（最小化缓冲）作为进一步优化
