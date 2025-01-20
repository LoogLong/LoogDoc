# 主要流程
1. 读取动画姿势  masterJob = bm.ReadTransform(masterJob);
    使用动画姿势来更新 bone manager 中的buffer
    input：动画姿势buffer
    output：bm.positionArray, bm.rotationArray，bm.localPositionArray, bm.localRotationArray, bm.localToWorldMatrixArray

2. 将动画姿势应用给代理网格。作为代理网格的基本姿势  masterJob = vm.PreProxyMeshUpdate(masterJob);
    使用bone manager中的transform信息，更新代理网格
    input: bm.localToWorldMatrixArray
    output: vm.positions, rotations

3. 更新team的中心和惯性 masterJob = tm.CalcCenterAndInertiaAndWind(masterJob);
    使用virtual mesh manager和bone manager中的transform计算team的中心和惯性
    input: vm.positions,vm.rotations. bm.positionArray, bm.rotationArray
    outut: TeamDataArray, TeamCenterData, WindData

4. 重置所有质点或者处理因team中心移动带来的惯性 masterJob = sm.PreSimulationUpdate(masterJob);
    如果是重置：根据代理网格(virtual mesh manager)中的顶点位置和旋转，直接更新simulation manager中的buffer
        input: vm.positions, vm.rotationos
        output: sm中所有buffer
    如果是惯性：根据sm中的buff加上CenterData，更新sm中的buffer
        input:Team.teamDataArray，Team.parameterArray，Team.centerDataArray
        output: sm中的所有buffer

5. 更新碰撞器 masterJob = MagicaManager.Collider.PreSimulationUpdate(masterJob);
    根据bone manager中的transform信息更新collider manager
    input：bm.positionArray, bm.rotationArray
    output: collider manager中的 old和new transform

6. 执行布料模拟 ： masterJob = sm.SimulationStepUpdate(maxUpdateCount, i, masterJob);
    a. jobHandle = tm.SimulationStepTeamUpdate(updateIndex, jobHandle);
    b. 拆分 ParticleJob？CreateUpdateParticleList
    c. collider 更新CreateUpdateColliderList / StartSimulationStep
    d. 质点更新 StartSimulationStepJob
        使用virtual mesh和team manager作为输入。计算 particle的next transform和base Transform。base transform有两个，一个是ref pose，一个是anim pose，可以在这两者之间差值
        input:vm.positions, vm.rotations, team.TeamDataArray,team.centerDataArray,team.parameterArray
        output:sm.NextPosArray, sm.BasePosArray,sm.BaseRotArray,sm.VelocityPosArray,sm.StepBasicPositionBuffer, StepBasicRotationBuffer
    e. 计算约束求解的基准姿势。根据参数，混合ref pose和anim pose：UpdateStepBasicPotureJob
        根据virtual mesh中的local transform信息更新ref pose，并与anim pose做混合,并输出混合后的姿势
        input：sm.BasePosArray,sm.BaseRotArray, sm.StepBasicPositionBuffer,sm.StepBasicRotationBuffer
        output：sm.StepBasicPositionBuffer,sm.StepBasicRotationBuffer
    f. 解约束:
        input：sm.NextPosArray
        1. Tether约束，root与其链上每个particle的约束。
        2. Distance约束，包括竖直方向的结构约束，剪切约束，横向的距离约束
        3. AngleConstraint，摆动的主要来源，包括了角度限制和角度恢复：
            角度限制，是在链上，将父子两个质点之间限制角度。并且是从父到子迭代
        4. triangle bending
        5. 碰撞检查。
        6. 再来一次distance constraint
        7. motion constraint，最大距离限制
        8. 自碰撞
    g. EndSimulationStepJob
        根据前面计算的simulation manager中的NextPosArray中的数据，计算真实速度。并将最后的质点位置记录在sm.OldPosArray中

7. 模拟完成后，使用代理网格位置驱动显式位置：masterJob = sm.CalcDisplayPosition(masterJob);
    根据virtual meah manager中的transform，计算simulation manager中的display pos array.
    根据simulation manager中的OldPosArray，修改virtual mesh manager中的positions
    input: sm.OldPosArray，vm.positions, vm.rotations
    output: sm.DispPosArray，vm.positions

8. 布料模拟后，计算质点的姿势。如果是链式，则会针对链条，调整姿势。对于bone cloth 将顶点姿势复制到对应的transform中  masterJob = vm.PostProxyMeshUpdate(masterJob);
    根据virtual mesh manager中的 顶点transform以及父子信息计算 vm中的顶点的rotation
    input：vm中的顶点transform,以及其父子关系
    output: vm的rotation

9. 将transform写入动画masterJob = bm.WriteTransform(masterJob);
    将bone manager中的transform写入动画姿势
    input：bm.positionArray, bm.rotationArray,bm.localPositionArray,bm.localRotationArray
    output：动画姿势buffer


# 速度计算
有专用的速度位置。
速度位置更新流程：
1. 读取上一次的最终位置作为速度位置的起始点。
2. 施加惯性偏移
3. 约束与碰撞修改
4. 计算速度


