# OpenClaw 内网部署 - 网络白名单

企业内网环境下，需要在防火墙/代理上开放以下外部域名的出站访问（HTTPS 443）。

---

## 1. 包管理源

### npm (Node.js)
| 域名 | 用途 |
|------|------|
| `registry.npmjs.org` | npm 包安装、版本查询 |
| `www.npmjs.com` | npm 包信息页面 |

### PyPI (Python)
| 域名 | 用途 |
|------|------|
| `pypi.org` | Python 包索引 |
| `files.pythonhosted.org` | Python 包下载 |

### Go
| 域名 | 用途 |
|------|------|
| `proxy.golang.org` | Go 模块代理 |
| `sum.golang.org` | Go 模块校验和 |
| `storage.googleapis.com` | Go 模块存储 |

### Maven / Gradle (Java)
| 域名 | 用途 |
|------|------|
| `repo.maven.apache.org` | Maven Central |
| `repo1.maven.org` | Maven Central 镜像 |
| `plugins.gradle.org` | Gradle 插件仓库 |
| `services.gradle.org` | Gradle Wrapper 下载 |

### Cargo (Rust)
| 域名 | 用途 |
|------|------|
| `crates.io` | Rust 包索引 |
| `static.crates.io` | Rust 包下载 |
| `index.crates.io` | Sparse 索引 |

### RubyGems (Ruby)
| 域名 | 用途 |
|------|------|
| `rubygems.org` | Ruby 包安装 |

### NuGet (.NET)
| 域名 | 用途 |
|------|------|
| `api.nuget.org` | NuGet 包索引和下载 |

---

## 2. 容器镜像仓库

| 域名 | 用途 |
|------|------|
| `registry-1.docker.io` | Docker Hub 镜像拉取 |
| `production.cloudflare.docker.com` | Docker Hub CDN |
| `auth.docker.io` | Docker Hub 认证 |
| `ghcr.io` | GitHub Container Registry |
| `registry.k8s.io` | Kubernetes 官方镜像 |
| `quay.io` | Red Hat Quay 镜像 |
| `*.ecr.amazonaws.com` | AWS ECR（如使用） |
| `*.azurecr.io` | Azure ACR（如使用） |

---

## 3. 代码托管 / Git

| 域名 | 用途 |
|------|------|
| `github.com` | Git clone/push、Release 下载 |
| `api.github.com` | GitHub API |
| `raw.githubusercontent.com` | GitHub 原始文件 |
| `objects.githubusercontent.com` | GitHub LFS / Release Assets |
| `gitlab.com` | GitLab（如使用） |
| `bitbucket.org` | Bitbucket（如使用） |

---

## 4. 系统工具 / 运行时安装

| 域名 | 用途 |
|------|------|
| `nodejs.org` | Node.js 安装包 |
| `bun.sh` | Bun 运行时安装 |
| `deb.nodesource.com` | Node.js APT 源 |
| `dl.yarnpkg.com` | Yarn 安装 |
| `www.python.org` | Python 安装包 |
| `download.docker.com` | Docker Engine 安装 |
| `get.docker.com` | Docker 安装脚本 |
| `dl.k8s.io` | kubectl 等 K8s 工具 |
| `apt.kubernetes.io` | K8s APT 源 |
| `helm.sh` / `get.helm.sh` | Helm 安装 |
| `raw.githubusercontent.com` | 各类安装脚本托管 |

---

## 5. OS 包管理（Linux）

### Debian / Ubuntu (APT)
| 域名 | 用途 |
|------|------|
| `deb.debian.org` | Debian 官方源 |
| `archive.ubuntu.com` | Ubuntu 官方源 |
| `security.debian.org` | Debian 安全更新 |
| `security.ubuntu.com` | Ubuntu 安全更新 |
| `ppa.launchpadcontent.net` | Ubuntu PPA |

### RHEL / CentOS (YUM/DNF)
| 域名 | 用途 |
|------|------|
| `mirrorlist.centos.org` | CentOS 镜像列表 |
| `vault.centos.org` | CentOS 存档源 |
| `cdn-ubi.redhat.com` | RHEL UBI 源 |

### Alpine (APK)
| 域名 | 用途 |
|------|------|
| `dl-cdn.alpinelinux.org` | Alpine 包源 |

---

## 6. 证书 / 安全验证

| 域名 | 用途 |
|------|------|
| `ocsp.digicert.com` | OCSP 证书状态查询 |
| `ocsp.pki.goog` | Google OCSP |
| `crl.microsoft.com` | Microsoft CRL |
| `keyserver.ubuntu.com` | GPG 公钥服务器 |
| `keys.openpgp.org` | OpenPGP 公钥 |

---

## 7. 国内镜像源（可选替代）

如果无法直接访问国际源，可配置国内镜像：

### npm 镜像
```bash
npm config set registry https://registry.npmmirror.com
```
| 域名 | 用途 |
|------|------|
| `registry.npmmirror.com` | 淘宝 npm 镜像 |
| `cdn.npmmirror.com` | npmmirror CDN |

### PyPI 镜像
```bash
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```
| 域名 | 用途 |
|------|------|
| `pypi.tuna.tsinghua.edu.cn` | 清华 PyPI 镜像 |
| `mirrors.aliyun.com` | 阿里云 PyPI 镜像 |

### Docker 镜像
```json
// /etc/docker/daemon.json
{ "registry-mirrors": ["https://mirror.ccs.tencentyun.com"] }
```
| 域名 | 用途 |
|------|------|
| `mirror.ccs.tencentyun.com` | 腾讯云 Docker 镜像 |
| `registry.cn-hangzhou.aliyuncs.com` | 阿里云 Docker 镜像 |

### Go 镜像
```bash
go env -w GOPROXY=https://goproxy.cn,direct
```
| 域名 | 用途 |
|------|------|
| `goproxy.cn` | 七牛 Go 代理 |

### Maven 镜像
| 域名 | 用途 |
|------|------|
| `maven.aliyun.com` | 阿里云 Maven 镜像 |

---

## 快速白名单汇总

按优先级排列，复制到防火墙规则：

```
# ===== 必须 =====
# npm（Node.js 项目核心依赖）
registry.npmjs.org

# 容器镜像（K8s 部署）
registry-1.docker.io
production.cloudflare.docker.com
auth.docker.io
ghcr.io
registry.k8s.io

# Git
github.com
api.github.com
raw.githubusercontent.com
objects.githubusercontent.com

# ===== 按需 =====
# Python
pypi.org
files.pythonhosted.org

# Go
proxy.golang.org
sum.golang.org

# Java
repo.maven.apache.org

# Rust
crates.io
static.crates.io

# 运行时安装
nodejs.org
bun.sh
download.docker.com
get.helm.sh
dl.k8s.io

# OS 包管理（根据基础镜像选择）
deb.debian.org
security.debian.org
dl-cdn.alpinelinux.org
```

---

## 注意事项

1. **协议**：所有域名均使用 HTTPS（TCP 443），部分还需 HTTP（TCP 80）用于重定向
2. **DNS**：确保内网 DNS 能解析以上域名，或配置 DNS 转发到公网 DNS
3. **代理配置**：可通过 `HTTP_PROXY` / `HTTPS_PROXY` 环境变量统一走 HTTP 代理，免逐条开放
4. **国内镜像**：如果直连国际源较慢或不通，优先使用第 7 节的国内镜像替代
5. **容器构建**：Docker 多阶段构建中 `npm install` 需要同时开放 npm 和 GitHub（部分包从 GitHub 拉取）
6. **GPG 密钥**：添加 APT/YUM 源时需访问 `keyserver.ubuntu.com` 等密钥服务器验证签名
