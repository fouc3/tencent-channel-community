# 本地打包说明（tencent-channel-cli-bin）

上游 CLI 只发 npm 预编译二进制、无源码仓库，所以这里是**预编译二进制重打包**，按 Arch 的 `-bin` 惯例命名。

## 构建

```bash
cd packaging
# 上游 tarball 会自动从 registry.npmjs.org 下载（sha256 见 PKGBUILD 的 sha256sums）
makepkg -f
# 产物形如 tencent-channel-cli-bin-1.0.10-1-x86_64.pkg.tar.zst
namcap tencent-channel-cli-bin-*.pkg.tar.zst   # 预期只剩 PIE / FULL RELRO 两条上游固有警告
```

## 安装

```bash
sudo pacman -U tencent-channel-cli-bin-1.0.10-1-x86_64.pkg.tar.zst
pacman -Q tencent-channel-cli-bin
```

## 升级到上游新版

```bash
npm view tencent-channel-cli version      # 例如 1.0.11
npm pack tencent-channel-cli-linux-x64@<新版本>   # 取新 tarball 并算 sha256
sha256sum tencent-channel-cli-linux-x64-<新版本>.tgz
# 改 PKGBUILD：pkgver、sha256sums（Package 内的 PROVENANCE 版本号也一并改）
makepkg -f && sudo pacman -U tencent-channel-cli-bin-<新版本>-1-x86_64.pkg.tar.zst
```

## 说明

- 只安装 npm 平台包 `tencent-channel-cli-linux-x64` 内的 `bin/tencent-channel-cli`（Go 静态链接、无运行时依赖）；npm 元包 `tencent-channel-cli` 只是 node 转发脚本，本机不需要。
- 许可证：上游未声明，按 `LicenseRef-custom`，随包安装 `/usr/share/licenses/tencent-channel-cli-bin/LICENSE`。
- 本机 `/tmp` 是内存盘，packaging/ 里的 PKGBUILD 是持久副本；重新构建即可再生成包。
