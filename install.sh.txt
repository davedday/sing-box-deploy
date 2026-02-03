#!/bin/bash
set -e

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

print_info() { echo -e "${GREEN}[INFO]${NC} $1"; }
print_error() { echo -e "${RED}[ERROR]${NC} $1"; }
print_warning() { echo -e "${YELLOW}[WARNING]${NC} $1"; }

[[ $EUID -ne 0 ]] && print_error "需要 root 权限运行" && exit 1

print_info "=== Sing-box 一键部署 ==="

# 安装 Docker
if ! command -v docker &> /dev/null; then
    print_info "安装 Docker..."
    curl -fsSL https://get.docker.com | sh
    systemctl start docker && systemctl enable docker
fi

# 安装 openssl
if ! command -v openssl &> /dev/null; then
    apt update && apt install -y openssl
fi

# 删除旧容器
docker rm -f sing-box 2>/dev/null || true

# 创建配置目录
mkdir -p /etc/sing-box

# 下载加密配置
print_info "下载配置..."
curl -fsSL https://raw.githubusercontent.com/davedday/sing-box-deploy/main/config.json.enc -o /tmp/config.json.enc

# 解密
MAX_ATTEMPTS=3
ATTEMPT=0

while [ $ATTEMPT -lt $MAX_ATTEMPTS ]; do
    print_warning "请输入配置密码:"
    read -s PASSWORD
    echo ""
    
    if openssl enc -aes-256-cbc -d -pbkdf2 -in /tmp/config.json.enc -out /etc/sing-box/config.json -pass pass:$PASSWORD 2>/dev/null; then
        print_info "✅ 配置解密成功"
        rm /tmp/config.json.enc
        break
    else
        ATTEMPT=$((ATTEMPT + 1))
        REMAINING=$((MAX_ATTEMPTS - ATTEMPT))
        [ $REMAINING -gt 0 ] && print_error "密码错误，还有 $REMAINING 次机会" || { print_error "密码错误次数过多"; rm /tmp/config.json.enc; exit 1; }
    fi
done

# 启动
print_info "启动容器..."
docker run -d --name sing-box --restart always \
  -v /etc/sing-box:/etc/sing-box \
  -p 20000:20000/tcp -p 20000:20000/udp \
  ghcr.io/sagernet/sing-box:latest \
  -D /var/lib/sing-box -C /etc/sing-box run

sleep 3

if docker ps | grep -q sing-box; then
    print_info "✅ 部署成功！端口: 20000"
    echo "查看日志: docker logs -f sing-box"
    docker logs -f sing-box
else
    print_error "启动失败"
    exit 1
fi
