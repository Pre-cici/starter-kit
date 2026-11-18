# Starter Kit

开始之前请先阅读 [Introduction](/docs/Introduction.md)

## Layout

```text
smart_buildings/
  smart_control/            # 建筑仿真器（环境、仿真、奖励、模型、proto、工具）
  baselines/                # 用于对比的参考控制策略
  buildings/                # 训练集使用的 6 栋建筑定义
  weather/                  # 天气预设（6 个实测站点，训练集使用其中 5 个）
  training_scenarios/       # 30 个生成的场景（建筑 × 天气），各含 config.gin
  run_floor_generator.py    # 采样生成新的建筑配置
  run_scenario_generator.py # 建筑 × 天气 → 可运行场景
  run_environment.py        # 加载场景、检查空间、执行冒烟测试
  evaluate.py               # 在场景上评测你的策略（加载环境、运行轨迹、POLICY_MAP）
assets/                     # 文档使用的图片
```

## Quickstart

先安装[uv](https://docs.astral.sh/uv/getting-started/installation/)。

```bash
git clone https://github.com/Pre-cici/starter-kit.git
cd starter-kit

uv sync --locked --python 3.12 --extra dev
```

```bash
# 1. 查看一个场景：动作/观测空间及简短仿真轨迹
uv run python -m smart_buildings.run_environment --building building_train_1 --weather fairbanks_1

# 2. 在全部 30 个场景上评测初始策略
uv run python -m smart_buildings.evaluate
```

上述命令运行的是 `transfer`，即 `smart_buildings/baselines/transfer_policy.py` 中的初始策略。

### weather data and training set

仓库提供 **6 个天气站、48 个天气片段**。

- 六个天气站分别为 `fairbanks`、`denver`、`atlanta`、`phoenix`、`singapore` 和 `minneapolis`，每站有一份 [CSV 原始数据](smart_buildings/weather/csv/)
- 每个天气站有 8 份 [片段配置](smart_buildings/weather/)，文件名格式为 `<station>_<n>.yaml`，同站的片段共用一份 CSV。
- 通过 `start_timestamp` 指定从原始CSV中截取的连续 30 天。
- 仿真每 5 分钟推进一步，每个完整场景共 **8,640 步**。生成器不支持合成天气。

仓库发布的 **30 个训练场景**:

- 天气站只使用 `fairbanks`、`denver`、`atlanta`、`phoenix` 和 `singapore`
- 6 栋建筑各从这 5 个站选配一段天气，每个天气片段只用一次。
- 并非所有建筑和天气片段的组合都存在。
- 具体配对见 [training_set.yaml](smart_buildings/training_set.yaml)
- `minneapolis` 未被纳入发布的训练集，但可用于生成新场景。

在仓库根目录运行：

```bash
# 查看全部天气站和片段配置
ls smart_buildings/weather/csv/
ls smart_buildings/weather/*.yaml

# 查看 Fairbanks 各片段的开始时间
grep -H '^start_timestamp:' smart_buildings/weather/fairbanks_*.yaml

# 使用以下命令查看某栋建筑可用的天气片段：
ls smart_buildings/training_scenarios/building_train_1/
```

## Writing Your Policy

**编辑 `smart_buildings/baselines/transfer_policy.py`。** 该文件的模块文档字符串完整定义了接口规范，包括:

- 动作布局
- 观测布局
- 各设定值的物理作用及常见陷阱。

**应该阅读[[./docs/transfer_policy.md]], 而不是本节作为接口细节的权威参考**。

用于本地评测的策略是一个具有两个方法的普通 Python 类，**不需要继承任何基类**：

```python
class TransferPolicy:
    def __init__(self, building_meta_data, action_spec):
        # building_meta_data: dict with action indices, observation field names,
        #   zone-to-AHU assignment, floor plan, schedule, normalization stats, ...
        # action_spec: the hybrid action spec for this building
        ...

    def get_action(self, obs) -> dict:
        # return {"continuous_action": np.float32[3K+2+2V],
        #         "discrete_action":   np.int32[K+1]}
        ...
```

保留类名就无需更改其他内容，因为它已经注册在 `POLICY_MAP` 中。
如果重命名该类或者添加第二个策略，需要在 `POLICY_MAP` 中注册，并使用对应的键传给 `--policies`。

需要特别强调文档字符串中的两点：

1. **连续动作采用交错排列**
   - 形式为`[temp_0, pressure_0, recirc_0, temp_1,...]`，接着附加两个热水系统控制量，之后才是各 Zone 的控制量。
   - 应读取 `building_meta_data["temp_indices"]`、`["pressure_indices"]`、`["recirculation_indices"]`、`["water_indices"]` 和 `["zone_action_indices"]`，而不是假设固定偏移量。
   - 硬编码布局的策略可能看似正常运行，却悄悄控制了错误的执行器。

2. **零动作不等于关闭动作**
   - 对连续动作而言，零表示归一化区间的中点；因此，某个未赋值的位置可能意味着风阀半开，而不是关闭。
   - 对于需要控制的每个位置，包括希望保持静止或最低运行状态的位置，都应明确赋值。

### Evaluate Your Policy

```bash
# 一个场景：完整的 30 天 Episode
uv run python -m smart_buildings.evaluate --policies transfer \
    --buildings building_train_1 --weathers fairbanks_1

# 全部 30 个场景
uv run python -m smart_buildings.evaluate --policies transfer

# 冒烟测试：只运行一天，速度快，但不能与 Baseline 直接比较
uv run python -m smart_buildings.evaluate --policies transfer \
    --buildings building_train_1 --weathers fairbanks_1 --steps 288
```

- `--steps` 默认值为 8640，即完整的 30 天 Episode，与 `baseline_returns.json` 中基线评测所使用的长度相同。
- 只有进行冒烟测试时才应缩短时长：较短的仿真可能跳过节能回退（eco-setback）时段，并使考虑运行时间表的策略显得更好。
- **不要比较不同步数下获得的 Return**：较短 Episode 的总奖励通常负值更小，从而可能让较弱的策略看起来更好。

对于每个场景，评测结果写入 `smart_buildings/results/<building>/<weather>/<policy>/summary.json`，并包含图表和时间序列数据。

## Generating More Training Data

Buildings采用程序化方式生成，因此可以在开发阶段采样任意数量的新buildings和scenarios。
Held-out set 中的建筑也来自同一生成器及其参数范围。

```bash
# 根据种子 500–509 生成 10 栋建筑的 YAML 和 PDF 预览
# 若楼层平面图存在小于 20 m² 的 Zone，则跳过对应种子并在末尾列出
uv run python -m smart_buildings.run_floor_generator --start-seed 500 --count 10

# 将建筑 × 天气片段转换为可运行场景
uv run python -m smart_buildings.run_scenario_generator \
    --buildings building_0500 building_0501 --weathers fairbanks_1 phoenix_6

# 查看或绘制生成的场景
uv run python -m smart_buildings.run_environment --building building_0500 --weather fairbanks_1
uv run python -m smart_buildings.run_scenario_generator --plot building_0500 fairbanks_1
```

同时指定 `--buildings` 和 `--weathers` 时，会生成两个列表的**笛卡尔积**。
如果运行 `run_scenario_generator` 而不传参数，则会根据 `training_set.yaml` 重新构建发布的 30 个场景配对。

关于生成数据，还有两点需要注意：

- **Construction**: 每栋生成建筑都有独立的 `materials:` 配置块，其生成方式与发布的 6 栋建筑相同：
  - 在批次内独立采样并分层分布 6 种围护结构属性。
  - 参数范围及设计理由见 `smart_control/utils/materials.py`。
  - 使用 `--no-materials` 可以回退到一套默认建筑构造参数。
- **AHU Count**: YAML 中的 `num_ahus` 是下限，不一定等于最终数量。
  - 场景生成器会将它提高到至少 `ceil(zones / 10)`，因此建筑占地布局决定实际 AHU 数量。
  - 默认生成的建筑平面尺寸范围比发布的 6 栋建筑更大，所以生成建筑可能拥有更多 AHU，仿真成本也会相应增加。
  - 如果希望生成与发布训练集规模相近的建筑，可缩小 YAML 中的 `width` 和 `height`。

## Reference Baselines

除 `transfer` 外，`evaluate.py` 中的 `POLICY_MAP` 还提供以下五种策略，供你了解需要对比的控制方法：

| 键（key）        | 策略说明                                      |
| ---------------- | --------------------------------------------- |
| `bang_bang`      | 根据舒适温度区间，对每个 AHU 执行开关控制     |
| `fixed_setpoint` | 全天采用统一舒适温度范围，不进行节能回退      |
| `pid`            | 比例控制，风机压力随控制强度变化              |
| `pid_water`      | 在 `pid` 基础上增加再热调节及按需重设热水温度 |
| `all_off`        | 始终不运行任何设备                            |

在仓库根目录运行以下命令，在 `building_train_1` 的 `fairbanks_1` 场景上评估 `pid` 和 `bang_bang`。每种策略各跑完整的 8,640 步：

```bash
uv run python -m smart_buildings.evaluate \
  --policies pid bang_bang \
  --buildings building_train_1 \
  --weathers fairbanks_1 \
  --steps 8640 \
  --results-dir smart_buildings/results/baseline_example
```

如需评估其他基线，替换 `--policies` 后的策略名。若要评估默认场景目录中的所有场景，去掉 `--buildings` 和 `--weathers` 两行。

前四种策略与参赛策略一同作为成绩的参考区间。

`all_off` 用于观察完全不调节的建筑会产生多大代价，但不包含在上述参考区间中：彻底关闭所有设备并不是现实控制器会采用的策略，因此无法说明一个实际控制器表现如何。

跑完上面的评估后，在仓库根目录运行以下命令，查看 `pid` 在该场景的分数及四种基线的参考成绩：

```bash
uv run python score_leaderboard.py \
  smart_buildings/results/baseline_example \
  --policy pid \
  --expect 1
```

脚本默认从 `baseline_returns.json` 读取参考成绩，不需要在本地跑齐四种基线。这里只评估了一个场景，提示其他场景缺少 `pid` 结果是正常的；要查看 `bang_bang` 的分数，将 `--policy pid` 改为 `--policy bang_bang`。

**官方没有提供学习得到的基线策略（Learned Baseline），需要参赛者自行实现。**

## Scoring

每个场景都会报告策略的**总奖励（Total Reward）**，即在整个 Episode 中，对每步的人员舒适度、能源费用和碳排放加权结果进行求和。

排行榜采用的指标是**每个场景平均每步奖励相对于目标值零的平均绝对误差（Mean Absolute Error Against Zero）**。**分数越低越好。**

每一步奖励都不会为正：舒适度不足、能源费用和碳排放均产生惩罚，而不存在正向奖励。因此，绝对值相当于损失的大小。零分意味着完全满足舒适度需求，同时不消耗能源、不产生碳排放；现实中没有控制器能达到这种理想状态。可以将零视为理想目标，将评分视为与该目标的距离。

30 个场景的评分采用**等权平均**。不按建筑或天气先进行分组，因为分组会引入任意权重，并掩盖组内差异。

仓库根目录的 `baseline_returns.json` 提供了四种基线在全部 30 个发布训练场景上的总奖励，均使用 8640 步（30 天）Episode。
**这些基线值不参与最终评分计算**，因为评分指标本身不依赖基线。

`score_leaderboard.py` 只是把基线成绩显示在你的成绩旁边，作为开发参考，因此你只需运行自己的策略评测。

`evaluate` 默认就是 8640 步，可以直接用于评分。

不同 Episode 长度的 Return 不能比较，因此如果你的策略使用了其他步数，评分程序会直接拒绝计算。指定 `--baselines self` 可以从你自己的结果目录读取参考基线成绩，但需要你同时运行四种基线。

官方排名由 Kaggle 服务器在隐藏评测建筑上计算：
<https://www.kaggle.com/competitions/neurips-2026-smart-buildings/>

隐藏建筑与发布的六栋建筑来自同一生成器及参数范围；天气片段也采用相同方法从气象站历史观测数据中构建，但不会使用这里发布的任何片段。因此策略必须迁移到**从未见过的建筑和天气条件**，而不能记忆发布的训练集。

## Reference File Guide

| 文件|用途|
| - | - |
|`run_floor_generator.py`|采样生成建筑配置（`building_XXXX.yaml` 和 PDF 预览）|
|`run_scenario_generator.py`|将建筑 × 天气组合生成 `training_scenarios/`（`config.gin`）；可绘制场景的楼层平面图和 Zone 分布图|
|`run_environment.py`|加载单个场景，打印动作/观测规格和元数据，运行简短仿真|
|`evaluate.py`| 在指定场景上评测**你的**策略（`POLICY_MAP` 中的任意键）|
|`score_leaderboard.py`|将结果目录转换成单个排行榜分数，并同时显示参考基线；默认读取提供的 `baseline_returns.json`，`--baselines self` 则从自己的结果目录读取 |
|`baseline_returns.json`|训练场景中各基线在 8640 步 Episode 上的 `total_reward`；只用于显示参考区间，不参与最终评分|
|`check_scales.py`| 检查各场景奖励归一化分母是否符合预期的 `c x K`|
