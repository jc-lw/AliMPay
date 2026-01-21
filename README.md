# AliMPay - 支付宝码支付系统

一个基于支付宝转账码的自动化支付解决方案，支持经营码收款和转账码收款两种模式。

![751192321ae102a907612a9ae6307903.png](https://i.mji.rip/2025/07/15/751192321ae102a907612a9ae6307903.png)

## 特性

- 🚀 **自动监控**: 实时监控支付宝账单，自动检测支付状态
- 📱 **码支付**: 支持经营码收款和转账码收款
- ⏰ **智能超时**: 5分钟支付时限，超时订单自动清理
- 🔐 **安全可靠**: MD5签名验证，防止数据篡改
- 🎯 **协议兼容**: 100%兼容CodePay协议

## 快速配置

### 1. 环境要求

- PHP 7.4+
- Composer
- 支付宝开放平台应用

### 2. 安装步骤

```bash
# 下载项目
# 安装依赖
composer install

# 复制配置文件
cp config/alipay.example.php config/alipay.php
```
# CodePay 支付宝免签支付系统宝塔面板部署手册

**适用环境**：Linux (CentOS/Ubuntu/Debian) + 宝塔面板 (aaPanel/BT Panel)
**核心架构**：Nginx + PHP 8.1 + SQLite (无MySQL)

---

## 一、 环境准备 (Prerequisites)

在宝塔面板 (App Store) 中安装以下软件：

1.  **Nginx**: 1.22 或更高版本
2.  **PHP**: **8.1** (推荐) 或 7.4

### 1.1 配置 PHP 扩展
进入 **App Store -> PHP 8.1 -> Extensions**，安装以下必须扩展：
* `sqlite3` & `pdo_sqlite` (数据库核心)
* `bcmath` (金额高精度计算)
* `gd` (生成二维码)
* `fileinfo` (文件处理)
* `curl` (网络请求)

### 1.2 解除禁用函数 (关键)
为了让系统能在后台自动监控订单，必须允许 PHP 执行系统命令。
进入 **App Store -> PHP 8.1 -> Disabled functions**，**删除**以下函数：
* `exec`
* `shell_exec`
* `proc_open`
* `pcntl_exec`
* `pcntl_signal`
* `pcntl_alarm`

> **操作后请务必重启 PHP 服务！** (Service -> Restart)

---

## 二、 网站搭建与代码上传

1.  **创建网站**：
    * 在 **Website** 菜单添加新站点。
    * **Domain**: 填写你的域名（例如 `pay.example.com`）。
    * **Database**: 选择 `No Database` (本项目使用 SQLite 文件数据库)。
    * **PHP Version**: 选择 `PHP-81`。

2.  **上传代码**：
    * 进入网站根目录 `/www/wwwroot/你的域名/`。
    * 删除默认的 `index.html` / `404.html`。
    * 上传所有项目文件。确保 `src/`, `config/`, `api.php` 等都在根目录下。

---

## 三、 安装依赖与权限设置

### 3.1 安装 Composer 依赖
点击网站根目录的 **Terminal** 按钮，或者通过 SSH 进入目录执行：

```bash
cd /www/wwwroot/你的域名/
# 安装生产环境依赖
composer install --no-dev --optimize-autoloader

### 3.2 设置文件权限 (非常重要)
确保 Web 服务 (www用户) 能写入数据库和日志。
```
# 将所有文件拥有者改为 www
chown -R www:www /www/wwwroot/你的域名/

# 设置目录读写权限
chmod -R 755 /www/wwwroot/你的域名/
chmod -R 777 /www/wwwroot/你的域名/data
chmod -R 777 /www/wwwroot/你的域名/logs
chmod -R 777 /www/wwwroot/你的域名/config
chmod -R 777 /www/wwwroot/你的域名/qrcodes
```

### 3.3  Nginx 安全加固
为了防止 SQLite 数据库 (.db) 和配置文件被直接下载，必须修改 Nginx 配置。 进入 Website -> 点击域名 -> Config，在 server { ... } 块中添加：
```
# 禁止访问敏感目录
location ~ ^/(data|config|src|vendor|logs)/ {
    deny all;
    return 404;
}

# 禁止下载敏感文件类型
location ~ \.(db|lock|json|log)$ {
    deny all;
    return 404;
}

# 禁止访问 Composer 文件
location ~ /composer\.(json|lock)$ {
    deny all;
    return 404;
}
```
### 3.3  首次初始化
在 SSH 终端执行以下命令（模拟 Web 请求来触发初始化）：
```
# 替换为你的实际目录
cd /www/wwwroot/你的域名/

# 强制触发监控启动和数据库生成
php -r '$_GET["action"]="force-start"; require "health.php";'
```
### 3. 支付宝配置

#### 获取支付宝应用参数

1. 登录 [支付宝开放平台](https://open.alipay.com)
2. 创建"网页/移动应用"
3. 获取以下参数：
   - **应用ID**: 应用详情页的AppId
   - **应用私钥**: 使用密钥工具生成
   - **支付宝公钥**: 从平台获取
   - **用户ID**: 账户中心的账号ID

可以参考这个[文章](https://www.mazhifu.me/mpay/35.html)申请应用，一般都会有一个默认的 生成密钥即可


#### 配置文件设置

编辑 `config/alipay.php`：

```php
<?php
return [
    'app_id' => 'YOUR_APP_ID',                    // 应用ID
    'private_key' => 'YOUR_PRIVATE_KEY',          // 应用私钥
    'alipay_public_key' => 'YOUR_ALIPAY_PUBLIC_KEY', // 支付宝公钥
    'transfer_user_id' => 'YOUR_USER_ID',         // 支付宝用户ID
    
    // 其他配置保持默认即可
];
```

#### 获取商户密钥

首次运行需要获取系统分配的商户ID和密钥：

```bash
# 启动服务
# 访问健康检查，系统会自动生成商户配置
curl http://domain/health.php

# 查看生成的商户信息
cat config/codepay.json
```

**商户配置文件示例**：
```json
{
    "merchant_id": "1001123456789012",
    "merchant_key": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
    "created_at": "2024-01-01 12:00:00",
    "status": 1,
    "balance": "0.00",
    "rate": "96"
}
```

**重要**：请妥善保存 `merchant_id` (商户ID) 和 `merchant_key` （商户密钥） ，这是商户接入时必需的参数。

### 4. 启动服务 下方两个模式二选一


## 码支付模式

### 经营码收款（推荐）

> 目前支付宝放水，进入经营码申请页面填写商户信息时，返回，就会提示是否免填写开启经营码
> 如果实在没有此处使用收款码替代，或者使用下方转账方式

**特点**: 无需转账备注，通过金额+时间匹配订单

1. **上传经营码**：将支付宝经营码二维码保存为 `qrcode/business_qr.png`

2. **启用配置**：编辑 `config/alipay.php`
```php
'payment' => [
    'business_qr_mode' => [
        'enabled' => true,  // 启用经营码模式
    ]
]
```

**工作原理**：
- 相同金额的订单自动增加0.01元偏移（1.00→1.01→1.02...）
- 客户扫码支付对应金额
- 系统通过金额和时间匹配订单

### 转账收款（无需额外配置）

**工作原理**：
- 客户转账时填写订单号作为备注
- 系统监控账单，通过备注匹配订单

## 支付宝调用流程

### 创建支付订单

```php
// 发起支付请求
$params = [
    'pid' => '商户ID',
    'type' => 'alipay',
    'out_trade_no' => '订单号',
    'notify_url' => '通知地址',
    'return_url' => '返回地址', 
    'name' => '商品名称',
    'money' => '支付金额',
    'sign' => '签名'
];

// POST 到 /submit.php 或 /mapi.php
```

### 监控支付状态

系统会自动：

1. **查询账单**: 每30秒查询支付宝账单API
2. **匹配订单**: 根据模式匹配相应订单
3. **更新状态**: 自动更新订单为已支付
4. **发送通知**: 向商户notify_url发送支付成功通知

### 查询订单状态

```bash
# 查询单个订单
GET /api.php?act=order&pid=商户ID&out_trade_no=订单号

# 查询商户信息
GET /api.php?act=query&pid=商户ID&key=商户密钥
```

## 支付页面

访问 `/submit.php` 生成支付页面，包含：

- 订单信息展示
- 二维码显示
- 实时倒计时（5分钟）
- 支付状态检查

## 系统监控

### 健康检查

```bash
# 检查系统状态
curl http://domain/health.php
```

### 日志查看

```bash
# 查看实时日志
tail -f data/app.log
```

## 目录结构

```
alimpay/
├── api.php              # API接口
├── submit.php           # 支付页面
├── mapi.php            # 移动端API  
├── health.php          # 健康检查
├── qrcode.php          # 二维码访问
├── config/             # 配置文件
│   └── alipay.php     # 支付宝配置
├── src/Core/          # 核心类库
├── data/              # 数据存储
└── qrcode/            # 二维码文件
```

## 签名算法

使用MD5签名算法：

```php
// 1. 参数按键名升序排序
// 2. 拼接成 key1=value1&key2=value2 格式  
// 3. 末尾拼接商户密钥
// 4. 计算MD5值

$signStr = 'money=0.01&name=测试&out_trade_no=123&pid=1001';
$sign = md5($signStr . $merchantKey);
```

## 常见问题


### Q: 支付检测延迟多久？  
A: 通常在支付完成后30秒内检测到并发送通知。

### Q: 订单超时时间是多久？
A: 订单创建后5分钟内必须完成支付，超时自动删除。

### Q: 如何调试支付问题？
A: 查看 `data/app.log` 日志文件，使用健康检查接口排查。

### Q: 如何部署到生产环境？
A: 将项目部署到Web服务器，配置好支付宝参数即可。


## 开源协议

MIT License

## 免责声明

本项目仅供学习交流使用，使用者需确保遵守相关法律法规和支付宝服务协议。 


