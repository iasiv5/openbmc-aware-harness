# USER.md - 你的人类

_描述这个仓库默认服务的对象。随着真实使用再收敛。_

- **默认服务对象：** OpenBMC firmware 开发者 / 团队成员
- **默认称呼：** 同学，或直接"你"
- **默认时区：** 北京（CST, UTC+8）

## 背景

**核心身份：**
- 固件与平台软件工程师
- 维护 OpenBMC layer、recipe、service、接口和平台配置的人
- 需要用 AI 加速需求分析、实现、调试和验证的人

**常见任务：**
1. 判断问题属于 common、platform、customer 还是 project。
2. 追踪 machine、layer、recipe、service、D-Bus object、Redfish route、IPMI 命令或 PLDM 流程。
3. 修改 bitbake recipe、patch、YAML schema、配置文件、unit 文件和测试。
4. 排查 build 失败、启动异常、runtime log、接口不通和平台差异。

**沟通偏好：**
- 先给结论，再给证据和展开。
- 用真实路径、服务名、接口名、错误码和日志说话。
- 明确影响范围，尤其是 common / platform / customer / project 边界。
- 没有验证就明确写“未验证”，不要暗示已经通过。

**会让他烦的：**
- 把 OpenBMC 问题说成笼统的“系统问题”或“后端问题”。
- 不区分层级边界就直接给改法。
- 没跑验证就说“应该可以”。
- 用空泛术语代替 recipe、service、D-Bus、Redfish 这类真实对象。

**成功协作的默认方式：**
- 先确认边界和归属。
- 再给最小改动方案。
- 最后给可执行验证命令和预期结果。