# 项目记忆

## 硬性约束
- 本地机器是 AMD 200GE（双核 4 线程 / 8G RAM），**绝不能在本地编译内核**（需 4 小时）。内核只在 CI 编译。
- CI 每轮全量 ~48 分钟（内核）+ 约 1 小时总耗时。**禁止凭感觉估时间**，实际以 gh 查询为准。
- 用户明确要求：不要过度思考；按时汇报进展；办事要靠谱，别浪费用户时间。

## pmbootstrap 关键知识
- 签名密钥是**每台机器随机生成**（abuild-keygen，PACKAGER=pmos），存在 `$WORK/config_abuild/`；公钥进 `$WORK/config_apk_keys/`。
- apk 包内的 `.SIGN.RSA.<key>.rsa.pub` 是**签名数据，不是公钥**。没有 config_abuild/config_apk_keys 就无法跨机器信任包。
- 跨 run 复用必须同时缓存：`packages/` + `config_abuild/` + `config_apk_keys/` + ccache，且 save 用 `if: always()`（失败 job 默认不存缓存）。
- `flasher export` 已移除；导出用 `pmbootstrap export <dir>`。

## CI 分工
- `build.yml`：唯一允许编译内核的工作流。always-save 缓存。
- `reuse.yml`：专职复用，`build_pkgs_on_install=false`，**禁止编译**，缺包直接报错。从缓存恢复，不从 artifact 恢复。
- build.yml 的 `paths-ignore` 含 `.github/workflows/**`，只改 workflow 不会触发编译。
- 设备：oneplus-instantnoodle（SM8250，fastboot）。firmware 包不得含 regulatory.db*（与 wireless-regdb 冲突）。

## 进行中
- CI run 31928200442 全量编译内核（05:05 启动，~48 分钟）。
- 刷机验证流程：adb root 备份分区 → fastboot 刷 boot/super/dtbo → USB 网卡 SSH root@172.16.42.1。
