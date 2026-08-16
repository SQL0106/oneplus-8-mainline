# 项目记忆

## 硬性约束
- 本地机器是 AMD 200GE（双核 4 线程 / 8G RAM），**绝不能在本地编译内核**（需 4 小时）。内核只在 CI 编译。
- CI 每轮全量 ~48 分钟（内核）+ 约 1 小时总耗时。**禁止凭感觉估时间**，实际以 gh 查询为准。
- 用户明确要求：不要过度思考；按时汇报进展；办事要靠谱，别浪费用户时间。
- 四条铁律（每个环节必须满足）：L-每步有 log；C-每步产物留档（本地+云端）；V-用真实数据验证（API/文件/magic/SHA256，绿≠通过）；R-内容可被后续复用。

## pmbootstrap 关键知识
- 签名密钥：只要 `$WORK/config_abuild/abuild.conf` 存在就**跳过随机密钥生成**（pmb/build/init.py:51）。固定密钥入仓 `ci/keys/`（pmos@local-ci.rsa + .rsa.pub + abuild.conf），workflow init 后复制到 config_abuild/ + config_apk_keys/。
- `$WORK/config_apk_keys` bind-mount 为 chroot `/etc/apk/keys`（pmb/config/__init__.py:174），预置公钥即可让 apk 信任。
- apk 包内 `.SIGN.RSA.<key>.rsa.pub` 是**签名数据，不是公钥**。
- 跨 run 复用依赖固定密钥（ci/keys/）+ actions cache（packages+config_abuild+config_apk_keys+ccache）。
- **actions/cache/save 失败只降级为 warning、步骤仍绿** → 必须用 curl caches API 自验证（需要 `actions: write` 权限）。
- `actions/cache/restore` 用 key+restore-keys 前缀匹配；恢复后必须验证包存在（缺则直接红）。
- `flasher export` 已移除；导出用 `pmbootstrap export <dir>`。export 的 `dtbs/` 是目录符号链接，`cp -L` 不带 `-r` 会丢弃它 → 用 `tar -ch` 全量包 + `cp -aL`。
- `pmbootstrap config build_pkgs_on_install false` 可让 install 零编译（config.py:58）。

## CI 分工
- `build.yml`：唯一允许编译内核的工作流。固定密钥注入 → 全量编译 → 缓存自验证 → 完整导出（含 dtbs）→ 上传产物 + pmos-packages + build-logs。
- `reuse.yml`：专职复用，`build_pkgs_on_install=false`，**禁止编译**，缺包直接报错。固定密钥注入 + 缓存恢复验证。
- build.yml 的 `paths-ignore` 含 `.github/workflows/**`，但 `ci/keys/` 等路径改动会触发编译，提交前注意。
- 设备：oneplus-instantnoodle（SM8250，fastboot）。firmware 包不得含 regulatory.db*（与 wireless-regdb 冲突）。
- 刷机：fastboot 刷 boot/dtbo/super。产物在 artifact `postmarketos-oneplus-instantnoodle`（含 boot.img/dtbo.img/rootfs/initramfs/vmlinuz/dtbs/export.tar）。

## 进行中 / 状态
- CI run 31931533225（固定密钥完整版）06:29 启动，预计 ~07:15 绿。
- 本地归档：`release/`（archive-packages/ 含历史所有 run 的 apk；logs/exec.log 执行日志）。
- 密钥：`ci/keys/pmos@local-ci.rsa`（私钥已入仓，勿外泄）。
