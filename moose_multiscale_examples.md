# MOOSE 中可供学习的 Multi-scale 案例汇总

面向需求：**热障涂层（TBC）热-力-化耦合损伤的多尺度计算**。

本文档整理了 MOOSE 仓库中与"多尺度"相关的功能模块、测试案例、示例和教程，
并按照与 TBC 多尺度建模的相关程度分类，方便按图索骥学习。

---

## 1. 跨尺度耦合框架：MultiApp 系统（框架层，最通用）

MOOSE 的多尺度/多物理耦合的底层机制是 `MultiApp`（主-子应用 + `Transfers` 数据传递），
可以做到"宏观(main app) ←→ 微观/介观(sub app)"的双向耦合，支持多级嵌套（多尺度桥接）。

- 文档：`framework/doc/content/syntax/MultiApps/index.md`（明确指出 MultiApp 是为
  multiscale 系统设计的："multiscale systems are generally loosely coupled between scales"）
- 官方教程（强烈推荐先学）：`tutorials/tutorial02_multiapps/`
  - `step01_multiapps/`：最基础的主-子应用搭建
  - `step02_transfers/04_parent_multiscale.i` 与 `04_sub_multiscale.i`：
    专门演示"多尺度(multiscale)"数据传递模式的例子
  - `step03_coupling/`：主子应用双向耦合（Picard/Fixed-Point 迭代）
  - 配套讲义：`doc/content/.../tutorial02_multiapps/presentation/step01_multiapps.md`、
    `step02_transfers.md`
- 相关综合教程：
  - `tutorials/darcy_thermo_mech/doc/content/workshop/systems/multiapps.md`、`problem/step10.md`
  - `tutorials/shield_multiphysics/doc/content/user_workshop/problem/step11.md`
  - `tutorials/user_short_workshop/doc/content/user_short_workshop/step6_coupling.md`

> 对 TBC 的意义：可以用宏观热-力有限元模型作为 main app，微观 RVE（含孔隙/裂纹/氧化层的
> 代表性体积单元）作为 sub app，通过 `Transfers` 把宏观应变/温度传给微观模型，再把微观均匀化
> 后的等效应力、损伤变量、有效导热系数传回宏观，形成 FE² 风格的多尺度求解。

---

## 2. 均匀化(Homogenization)系统——最直接可用于"由微观到宏观"的多尺度案例

### 2.1 渐近展开均匀化 (Asymptotic Expansion Homogenization, AEH)

**热传导（可直接用于 TBC 陶瓷层/多孔结构的等效热导率计算）**
- 内核/后处理源码：`modules/heat_transfer/src/kernels/HomogenizedHeatConduction.C`、
  `AnisoHomogenizedHeatConduction.C`；`modules/heat_transfer/src/postprocessors/HomogenizedThermalConductivity.C`
- 测试案例目录：`modules/heat_transfer/test/tests/homogenization/`
  - `heatConduction2D.i`：各向同性单胞均匀化热导率
  - `heatConduction2D_tensor_tc.i`：给定张量热导率的均匀化
  - `homogenize_tc_hex.i`：六边形（蜂窝状/纤维排布）几何的均匀化
- 文档：`HomogenizedHeatConduction.md`、`AnisoHomogenizedHeatConduction.md`、
  `HomogenizedThermalConductivity.md`（均在 `modules/heat_transfer/doc/content/source/...`）

**力学（等效弹性常数，可用于 TBC 陶瓷层/粘结层的等效力学性能）**
- 内核/后处理源码：`modules/solid_mechanics/src/kernels/AsymptoticExpansionHomogenizationKernel.C`、
  `modules/solid_mechanics/src/postprocessors/AsymptoticExpansionHomogenizationElasticConstants.C`
- 文档：`modules/solid_mechanics/doc/content/source/kernels/AsymptoticExpansionHomogenizationKernel.md`、
  `.../postprocessors/AsymptoticExpansionHomogenizationElasticConstants.md`

**综合示例（combined 模块，热+弹性质均匀化放在一起）**
- `modules/combined/examples/effective_properties/effective_th_cond.i`（等效热导率示例）

### 2.2 周期性单胞（RVE）应变/应力约束均匀化（Lagrangian Homogenization Constraint System）

面向大变形/损伤本构下的 RVE 均匀化，比 AEH 更适合非线性、损伤类问题（更贴近 TBC 损伤建模）。

- 理论与用法文档：`modules/solid_mechanics/doc/content/modules/solid_mechanics/Homogenization.md`
- 关键对象：
  - `ComputeHomogenizedLagrangianStrain`（材料）
  - `HomogenizedTotalLagrangianStressDivergence`（内核）
  - `AddPeriodicBCAction` + `SolidMechanics/QuasiStatic` Homogenization action
- 测试案例（从简单到复杂，建议按顺序学习）：
  - `modules/solid_mechanics/test/tests/lagrangian/cartesian/total/homogenization/small-tests/`
    （1d.i / 2d.i / 3d.i，小变形）
  - `.../homogenization/large-tests/`（大变形版本）
  - `.../homogenization/action/`（用 `SolidMechanics/QuasiStatic` action 简化输入的写法，
    含 2D 周期对称示例 `2d_pbc_symmetry`）
  - `.../homogenization/convergence/`（应力/应变加载对比收敛性研究）
  - `.../homogenization/residual_and_jacobian/3d.i`（残差/雅可比验证）
  - `.../homogenization/neml2/`（与 NEML2 高级本构库耦合的均匀化，`small_neml.i`、`large_neml.i`，
    若 TBC 陶瓷层/TGO 层损伤本构较复杂，可参考此耦合方式）
- 另有独立的短纤维/长纤维各向异性算例：
  `modules/solid_mechanics/test/tests/homogenization/anisoShortFiber.i`、`anisoLongFiber.i`
  （非常接近 TBC 层状/柱状晶结构的各向异性等效力学性能场景）

---

## 3. 结合微结构演化的多尺度示例（phase-field + mechanics）

TBC 涂层的氧化损伤、界面裂纹、晶粒长大等都与微结构演化耦合，以下示例展示"微观相场 + 力学"
耦合，可作为损伤/微结构演化尺度的参考：

- `modules/combined/examples/phase_field-mechanics/`
  - `kks_mechanics_KHS.i`、`kks_mechanics_VTS.i`：KKS 相场模型与力学耦合（多相/多组分，
    可类比 TGO 氧化层生长与应力耦合）
  - `EBSD_reconstruction_grain_growth_mech.i`：基于 EBSD 重构微结构的晶粒长大 + 力学
  - `interface_stress.i`：界面应力（可参考陶瓷层/粘结层界面损伤建模）
- `modules/combined/examples/periodic_strain/global_strain_pfm.i`、`global_strain_pfm_3D.i`：
  全局应变（均匀化外加载）驱动下的相场微结构演化，是"宏观加载 + 微观相场"耦合的直接范例

---

## 4. 化学-热-力耦合参考（面向 TBC 氧化/化学损伤）

TBC 高温服役中的热生长氧化层（TGO）涉及化学反应，以下模块可为"化"的部分提供参考：

- `modules/chemical_reactions/`：化学反应扩散、`solid_kinetics`、`aqueous_equilibrium` 等测试案例
- `modules/chemical_reactions/test/tests/thermochimica/`：与 Thermochimica 热力学平衡计算库耦合
  的案例（`MoRuPd.i`、`FeTiVO.i` 等），可参考其"MOOSE 场变量 <-> 外部化学热力学求解器"的耦合模式，
  用于 TBC 氧化反应动力学与温度场/应力场的耦合
- `modules/porous_flow/` 也包含反应输运（若 TBC 涉及气体/氧扩散通道，可参考其反应-输运实现）

---

## 5. 跨尺度 MultiApp 实战范例（结构最接近"宏观-微观"两个求解域）

`modules/porous_flow/examples/multiapp_fracture_flow/` 展示了"基质(matrix)"与"裂缝(fracture)"
两个不同尺度/维度的模型通过 MultiApp + Transfers 耦合求解，是学习"两套网格、两个尺度模型如何
互相传递场量"的很好范例（裂缝可类比 TBC 中的微裂纹/界面裂纹，基质可类比连续介质陶瓷层）：

- `fracture_diffusion/`：`matrix_app_dirac.i`、`matrix_app_nonconforming.i`、`fracture_app_dirac.i`
- `diffusion_multiapp/`：`fracture_app.i`、`fracture_app_heat.i`、`two_vars.i`（含热扩散）
- `3dFracture/matrix_app.i`：三维基质-裂缝耦合
- `single_fracture_heat_transfer/matrix_app.i`：单裂缝热传递耦合（结构上与"宏观热-力模型 + 微观
  裂纹/界面损伤模型"耦合思路一致）

---

## 6. 代理模型/降阶模型驱动的多尺度（可用于加速微观 RVE 反复求解）

多尺度计算中微观 RVE 通常需要重复求解、开销大，MOOSE 的 `stochastic_tools` 模块可训练代理模型
替代微观模型，从而加速宏微耦合迭代：

- `modules/combined/examples/stochastic/thermomech/`：`lhs_uniform.i`（拉丁超立方采样）+
  `poly_chaos_train_uniform.i`（多项式混沌代理模型训练），针对热-力问题
- `modules/combined/examples/stochastic/laser_welding_dimred/`：`train.i` / `test.i`，
  降维+代理建模范例
- 相关幻灯片：`modules/stochastic_tools/doc/stm_workshop/.../surrogate_workshop.md`

---

## 7. 建议的学习路径（针对 TBC 热-力-化耦合损伤多尺度计算）

1. 先学 `tutorials/tutorial02_multiapps/`，掌握 MultiApp + Transfers 的基本机制（尤其是
   `step02_transfers/04_parent_multiscale.i` / `04_sub_multiscale.i`）。
2. 学习均匀化系统：
   - 简单入门用 `modules/heat_transfer/test/tests/homogenization/`（等效热导率，AEH 方法）；
   - 力学部分学习 `modules/solid_mechanics/test/tests/lagrangian/.../homogenization/small-tests`
     到 `large-tests`，再到 `neml2/`（若后续本构较复杂，如含损伤演化）。
3. 参考 `modules/combined/examples/periodic_strain/` 和 `phase_field-mechanics/` 理解
   "宏观加载驱动的微结构/损伤演化"耦合写法。
4. 参考 `modules/porous_flow/examples/multiapp_fracture_flow/` 学习"两个不同尺度求解域通过
   MultiApp 双向传递场量"的具体写法，将其套用为"宏观 TBC 结构模型 + 微观 RVE（含裂纹/TGO/孔隙）
   损伤模型"。
5. 若需要引入氧化化学反应，参考 `modules/chemical_reactions/test/tests/thermochimica/` 的耦合模式，
   将化学反应/热力学平衡作为额外的（子）App 或材料模型嵌入到多尺度框架中。
6. 若微观 RVE 求解代价过高，参考 `modules/stochastic_tools` 与
   `modules/combined/examples/stochastic/thermomech/` 训练代理模型加速。

---

## 附：本文档中提到的关键路径速查表

| 主题 | 路径 |
| --- | --- |
| MultiApp 多尺度传输示例 | `tutorials/tutorial02_multiapps/step02_transfers/04_parent_multiscale.i`, `04_sub_multiscale.i` |
| 热导率渐近展开均匀化 | `modules/heat_transfer/test/tests/homogenization/` |
| 弹性常数渐近展开均匀化 | `modules/solid_mechanics/src/postprocessors/AsymptoticExpansionHomogenizationElasticConstants.C` |
| RVE 应力/应变约束均匀化（含大变形/NEML2） | `modules/solid_mechanics/test/tests/lagrangian/cartesian/total/homogenization/` |
| 各向异性纤维均匀化 | `modules/solid_mechanics/test/tests/homogenization/anisoShortFiber.i`, `anisoLongFiber.i` |
| 相场+力学（微结构-损伤演化） | `modules/combined/examples/phase_field-mechanics/` |
| 全局应变驱动的相场演化 | `modules/combined/examples/periodic_strain/` |
| 化学(氧化反应)-场耦合 | `modules/chemical_reactions/test/tests/thermochimica/` |
| 两尺度(基质-裂缝) MultiApp 耦合范例 | `modules/porous_flow/examples/multiapp_fracture_flow/` |
| 代理模型加速多尺度耦合 | `modules/combined/examples/stochastic/thermomech/`, `laser_welding_dimred/` |
| MultiApp 系统总文档 | `framework/doc/content/syntax/MultiApps/index.md` |
| 均匀化系统总文档 | `modules/solid_mechanics/doc/content/modules/solid_mechanics/Homogenization.md` |
