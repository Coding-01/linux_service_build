# 基本设置
```shell
# 换源
localhost:~# vi /etc/apk/repositories
#/media/cdrom/apks
# aliyunyuan                                   
http://mirrors.aliyun.com/alpine/v3.21/main          
http://mirrors.aliyun.com/alpine/v3.21/community     
# guanfangyuan                                       
http://dl-cdn.alpinelinux.org/alpine/v3.21/main      
#http://dl-cdn.alpinelinux.org/alpine/v3.21/community


localhost:~# apk update && apk add openssh vim bash bash-completion nginx curl net-tools

# 删除多个软件
localhost:~# apk del nginx mysql  



# Alpine为了精简，默认没装时区文件，必须手动安装
apk add tzdata
# 拷贝并设置中国时区
将时区设置为上海：
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo "Asia/Shanghai" > /etc/timezone


# 启动chrony并设置为开机自启
rc-service chronyd start
rc-update add chronyd default

# 如果你想立刻强制同步一次时间，执行：
chronyc -a makestep



在 Alpine 里，记住这两个命令：
rc-service [服务名] restart     替代 systemctl restart
rc-update add [服务名] default  替代 systemctl enable




# Alpine Linux服务管理
rc-update    # 主要用于不同运行级增加或者删除服务
rc-status    # 主要用于运行级的状态管理
rc-service   # 主用于管理服务的状态
openrc       # 主要用于管理不同的运行级


localhost:~# rc-service networking restart         # 重启网络服务
localhost:~# rc-status -a                          # 列出所有服务: 

localhost:~# apk add --no-cache openssh            # 不使用本地镜像源缓存，相当于先执行update，再执行add



# Apline网卡配置
# 配置DHCP
localhost:~# cat /etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp


# 配置静态IP
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
address 172.16.186.200
netmask 255.255.255.0
gateway 172.16.186.2
dns1 172.16.186.2
dns2 8.8.8.8




# 双网卡配置默认路由     暂略



```





# 在Alpine上部署yourls
```shell
Alpine 的特点是体积小、响应快，非常适合作为短链接跳转服务器

# 安装基础环境
localhost:~# apk add nginx mariadb mariadb-client \
php83-fpm php83-mysqli php83-curl php83-pdo_mysql \
php83-ctype php83-dom php83-gd php83-session php83-zlib \
php83-mbstring php83-bcmath php83-gettext php83-xml 
php83-simplexml  php83-curl


# 执行初始化设置
在 Alpine Linux 中，MariaDB 安装后并不会自动初始化数据库目录。需要手动执行初始化脚本
localhost:~# /etc/init.d/mariadb setup


启动服务并设置自启
初始化完成后，就可以正常启动了：
localhost:~# rc-service mariadb restart
localhost:~# rc-update add mariadb default

# 进行安全加固（设置root密码）
MariaDB默认安装后root用户是没有密码的，为了变现环境的安全，强烈建议执行：
localhost:~# mysql_secure_installation
Enter current password for root (enter for none):               # 没有密码直接回车
Switch to unix_socket authentication [Y/n] n
Change the root password? [Y/n] y
New password: 
Re-enter new password:                                          # 6a
Password updated successfully!

Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] n
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y



# 进入数据库创建yourls库
localhost:~# mysql -u root -p
CREATE DATABASE yourls;
CREATE USER 'yourlsuser'@'localhost' IDENTIFIED BY 'Aa7788**';
GRANT ALL PRIVILEGES ON yourls.* TO 'yourlsuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;





# 由于我是在 Alpine 上手动搭建，PHP 可能找不到 MariaDB 的套接字（socket）
localhost:~# ls -alh /run/mysqld/mysqld.sock
srwxrwxrwx    1 mysql    mysql          0 Apr 27 18:15 /run/mysqld/mysqld.sock

localhost:~# egrep ^'(pdo_mysql.default||mysqli.default)'_socket /etc/php83/php.ini 
pdo_mysql.default_socket=/run/mysqld/mysqld.sock
mysqli.default_socket = /run/mysqld/mysqld.sock



```





# 下载与配置 YOURLS
```shell
localhost:~# mkdir -p /var/www/yourls && cd /var/www/yourls
localhost:/var/www/yourls# wget https://github.com/YOURLS/YOURLS/archive/refs/tags/1.10.2.tar.gz
localhost:/var/www/yourls# tar -zxvf 1.10.2.tar.gz --strip-components=1


localhost:/var/www/yourls# touch /var/www/yourls/.htaccess


# 配置文件
localhost:/var/www/yourls# cp user/config-sample.php  user/config.php 
localhost:/var/www/yourls# vim user/config.php 
/** MySQL database username */
define( 'YOURLS_DB_USER', 'yourlsuser' );

/** MySQL database password */
define( 'YOURLS_DB_PASS', 'Aa7788**' );

/** The name of the database for YOURLS
 ** Use lower case letters [a-z], digits [0-9] and underscores [_] only */
define( 'YOURLS_DB_NAME', 'yourls' );

/** MySQL hostname.
 ** If using a non standard port, specify it like 'hostname:port', e.g. 'localhost:9999' or '127.0.0.1:666' */
define( 'YOURLS_DB_HOST', 'localhost' );

/** MySQL tables prefix
 ** YOURLS will create tables using this prefix (eg `yourls_url`, `yourls_options`, ...)
 ** Use lower case letters [a-z], digits [0-9] and underscores [_] only */
define( 'YOURLS_DB_PREFIX', 'yourls_' );

/** YOURLS网站地址 */
define( 'YOURLS_SITE', 'http://172.16.186.142' );          // 确保这里填的是你目前的访问 IP，末尾不要带斜杠

/** web端登录的帐号密码 */
$yourls_user_passwords = [
	'username' => 'password',        # 改成'admin' => '123456',
	// 'username2' => 'password2',   # 可添加多个帐号
	// You can have one or more 'login'=>'password' lines
];




# 配置Nginx(关键)
为了让短链接生效（即domain.com/abc跳转到真实地址），需要配置伪静态
localhost:/var/www/yourls# vim /etc/nginx/http.d/default.conf
原内容
server {
	listen 80 default_server;
	listen [::]:80 default_server;

	# Everything is a 404
	location / {
		return 404;
	}

	# You may need this to prevent return 404 recursion.
	location = /404.html {
		internal;
	}
}

改成
server {
    listen 80;
    server_name  172.16.186.142  127.0.0.1;
    root /var/www/yourls;
    index index.php index.html;

    # --- 关键：防盗链配置放这里 ---
    location /resources/ {
	    deny all;
        # 允许列出目录（如果需要），或者保持默认
        autoindex off;
    }

    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        include fastcgi.conf;
    }

    location / {
        try_files $uri $uri/ /yourls-loader.php$is_args$args;
    }

}




启动服务
localhost:/var/www/yourls# for i in nginx mariadb php-fpm83;do rc-service $i restart;done

# 设置开机自启
localhost:/var/www/yourls# for i in nginx mariadb php-fpm83;do rc-update add $i;done





# 测试数据库连接
localhost:/var/www/yourls# vim test_db.php
<?php
try {
    $pdo = new PDO('mysql:host=localhost;dbname=yourls', 'yourlsuser', 'Aa7788**');
    echo "连接成功！";
} catch (PDOException $e) {
    echo "连接失败: " . $e->getMessage();
}
?>


浏览器访问 http://172.16.186.142/test_db.php, 显示"连接成功"即配置正确




浏览器访问 http://172.16.186.142/admin 管理页面进行安装


```
<font color=red>**虽然安装完后它会自动失效，但为了安全请删除 rm /var/www/yourls/admin/install.php**</font>
![image]https://github.com/Coding-01/linux_service_build/blob/main/build_yourls/images/1.png)
![image]https://github.com/Coding-01/linux_service_build/blob/main/build_yourls/images/2.png)
![image]https://github.com/Coding-01/linux_service_build/blob/main/build_yourls/images/3.png)

# 获取 Signature Token
![image]https://github.com/Coding-01/linux_service_build/blob/main/build_yourls/images/4.png)
![image]https://github.com/Coding-01/linux_service_build/blob/main/build_yourls/images/5.png)












```shell
# 编写"资源下载页"(index.php)
在你的网站根目录（如 /var/www/yourls 或 /var/www/localhost/htdocs）下创建一个 index.php 文件
localhost:/var/www/yourls# vim /var/www/yourls/index.php
<?php
/**
 * CentOS 7 资源分发站 - 架构师随机令牌版
 * 安全性：每个文件验证码唯一、每日自动更换、防暴力破解
 */

date_default_timezone_set('Asia/Shanghai');

// --- 1. 配置区 ---
$server_ip = "172.16.186.142";
$yourls_api_url = "http://127.0.0.1/yourls-api.php"; 
$signature = "def05e4247"; 
$storage_path = __DIR__ . "/resources/"; 

// --- 2. 安全核心：加密盐值 (请务必修改下面这行，越复杂越好) ---
$secret_salt = "CentOS_RHCA_Secure_999"; 

/**
 * 核心算法：根据文件名和日期生成唯一随机码
 */
function generate_token($filename, $salt) {
    // 逻辑：将文件名、当天的日期、盐值混合哈希
    // 取 md5 后的前 6 位数字或字母作为提货码
    $hash = md5($filename . date("Ymd") . $salt);
    return strtoupper(substr($hash, 0, 6)); 
}

// --- 3. 下载拦截逻辑 ---
if (isset($_GET['download'])) {
    $file = basename($_GET['download']);
    $user_code = isset($_GET['code']) ? trim($_GET['code']) : '';
    
    // 实时计算该文件在今天的唯一合法 Token
    $valid_token = generate_token($file, $secret_salt);

    // 权限校验
    if (strcasecmp($user_code, $valid_token) !== 0) {
        header("Content-Type: text/html; charset=utf-8");
        die("<script>alert('授权码错误！\\n每个文件的验证码均不相同，且每日 0 点自动更换。'); window.location.href='index.php';</script>");
    }

    $full_path = $storage_path . $file;
    if (file_exists($full_path) && is_file($full_path)) {
        while (ob_get_level()) ob_end_clean();
        header('Content-Description: File Transfer');
        header('Content-Type: application/octet-stream');
        header('Content-Disposition: attachment; filename="' . $file . '"');
        header('Content-Length: ' . filesize($full_path));
        header('Cache-Control: no-cache');
        
        $handle = fopen($full_path, 'rb');
        while (!feof($handle)) {
            echo fread($handle, 32768);
            flush();
        }
        fclose($handle);
        exit;
    }
    die("文件不存在。");
}

$files_rpm = glob($storage_path . "*.rpm");
$files_mp4 = glob($storage_path . "*.mp4");

// 合并两个数组
$files = array_merge($files_rpm ?: [], $files_mp4 ?: []);
?>
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CentOS 7 企业级资源分发</title>
    <style>
        body { font-family: system-ui, -apple-system, sans-serif; background: #f4f7f6; margin: 0; padding: 20px; }
        .container { max-width: 850px; margin: 30px auto; background: #fff; padding: 30px; border-radius: 12px; box-shadow: 0 5px 25px rgba(0,0,0,0.05); }
        h1 { font-size: 22px; color: #1a1a1a; border-left: 5px solid #0056b3; padding-left: 15px; }
        .file-list { margin-top: 25px; border-top: 1px solid #eee; }
        .file-item { padding: 15px; border-bottom: 1px solid #f9f9f9; display: flex; align-items: center; justify-content: space-between; }
        .file-item:hover { background: #fcfcfc; }
        .file-info { color: #0056b3; font-weight: 600; cursor: pointer; text-decoration: none; }
        
        /* 弹窗样式 */
        #payModal { display:none; position:fixed; top:0; left:0; width:100%; height:100%; background:rgba(0,0,0,0.7); z-index:1000; backdrop-filter: blur(5px); }
        .modal-box { background:#fff; width:340px; margin:80px auto; padding:30px; border-radius:15px; text-align:center; }
        .qr-img { width: 200px; height: 200px; margin: 15px 0; border: 1px solid #eee; }
        input { width: 85%; padding: 12px; margin-bottom: 15px; border: 2px solid #0056b3; border-radius: 8px; text-align: center; font-size: 18px; font-family: monospace; }
        .dl-btn { background: #07c160; color:#fff; border:none; padding:12px 40px; border-radius:8px; cursor:pointer; font-weight:bold; width: 90%; }
    </style>
</head>
<body>

<div class="container">
    <h1>CentOS 7 官方资源仓库</h1>
    <p style="font-size:13px; color:#888;">安全审计已开启。每个资源需独立授权码，有效期至今日 24:00。</p>
    
    <div class="file-list">
        <?php foreach ($files as $file): $fn = basename($file); ?>
        <div class="file-item">
            <span>📦 <a class="file-info" onclick="askPay('<?php echo $fn; ?>')"><?php echo $fn; ?></a></span>
            <span style="font-size:12px; color:#999;"><?php echo round(filesize($file)/1024/1024, 2); ?> MB</span>
        </div>
        <?php endforeach; ?>
    </div>

    <div style="margin-top:30px; display:flex; align-items:center; background:#f9f9f9; padding:15px; border-radius:8px;">
        <img src="wx_code.jpg" style="width:80px; margin-right:15px;">
        <div style="font-size:12px; color:#555;">
            <strong>技术支持：资深 Linux 架构师</strong><br>
            支付后请扫码联系获取对应的 6 位提货码。<br>
            提示：不同文件的提货码不通用。
        </div>
    </div>
</div>

<div id="payModal">
    <div class="modal-box">
        <h3 id="targetName" style="font-size:16px; margin-top:0; white-space:nowrap; overflow:hidden; text-overflow:ellipsis;">授权验证</h3>
        <div style="color:#e4393c; font-size:20px; font-weight:bold;">￥5</div>
        <img src="pay_code.jpg" class="qr-img">
        <p style="font-size:12px; color:#666;">请支付后联系微信，发送文件名获取提货码</p>
        <input type="text" id="codeInput" placeholder="请输入 6 位随机码">
        <button class="dl-btn" onclick="submitDL()">立即验证并下载</button>
        <div style="margin-top:15px;"><a href="javascript:void(0)" onclick="hidePay()" style="color:#999; font-size:12px;">返回列表</a></div>
    </div>
</div>

<script>
let curFile = '';

function askPay(fn) {
    curFile = fn;
    document.getElementById('targetName').innerText = "授权对象：" + fn;
    document.getElementById('payModal').style.display = 'block';
}

function hidePay() {
    document.getElementById('payModal').style.display = 'none';
}

function submitDL() {
    const code = document.getElementById('codeInput').value.trim();
    if(!code) { alert("请输入提货码！"); return; }
    // 跳转下载
    window.location.href = `index.php?download=${encodeURIComponent(curFile)}&code=${encodeURIComponent(code)}`;
}
</script>

</body>
</html>







localhost:/var/www/yourls# nginx -t && rc-service nginx restart
localhost:/var/www/yourls# rc-service php-fpm83 restart






# 把所有rpm包放到这个目录中
localhost:/var/www/yourls# mkdir resources

# 在 Alpine Linux 中，Nginx 和 PHP-FPM 默认以 nginx 用户身份运行
localhost:/var/www/yourls# chown -R nginx:nginx /var/www/yourls/resources

localhost:/var/www/yourls# wget -O ./resources/rpm-4.19.1.1-20.el10.alma.1.x86_64.rpm \
https://repo.almalinux.org/almalinux/10/BaseOS/x86_64/os/Packages/rpm-4.19.1.1-20.el10.alma.1.x86_64.rpm



把pay_code.jpg、wx_code.jpg也都放到这个目录中



```
![image]https://github.com/Coding-01/linux_service_build/blob/main/build_yourls/images/6.png)
![image]https://github.com/Coding-01/linux_service_build/blob/main/build_yourls/images/7.png)






# 验证码管理页面
```shell
localhost:/var/www/yourls# cat code.php
<?php
/**
 * 架构师资源管理后台 - 提货码查询系统
 * 注意：必须以 <?php 开头，且前面不能有任何字符
 */

// --- 1. 安全防火墙 ---
// 自动检测并允许当前访问者的 IP，方便你第一次进入
// 建议：等你能成功打开页面后，再把下面的数组写死成固定 IP
$allowed_ips = [
    '127.0.0.1',
    '::1',
    '172.16.186.1', // 你的办公机 IP
];

// 如果你目前无法访问，请临时取消下面这一行的注释来查看你的真实 IP
// die("Your IP is: " . $_SERVER['REMOTE_ADDR']);

if (!in_array($_SERVER['REMOTE_ADDR'], $allowed_ips)) {
    header('HTTP/1.1 403 Forbidden');
    exit("<div style='color:red;padding:20px;'><h2>403 Access Denied</h2>你的 IP ({$_SERVER['REMOTE_ADDR']}) 不在授权白名单中。</div>");
}

// --- 2. 核心逻辑 ---
date_default_timezone_set('Asia/Shanghai');
$secret_salt = "CentOS_RHCA_Secure_999"; 
$storage_path = __DIR__ . "/resources/"; 

function get_token($filename, $salt) {
    $hash = md5($filename . date("Ymd") . $salt);
    return strtoupper(substr($hash, 0, 6));
}

//$files = glob($storage_path . "*.{rpm,mp4}", GLOB_BRACE);
// 方案：分别抓取 rpm 和 mp4，然后合并
$files_rpm = glob($storage_path . "*.rpm");
$files_mp4 = glob($storage_path . "*.mp4");

// 合并两个数组
$files = array_merge($files_rpm ?: [], $files_mp4 ?: []);

$search = isset($_GET['search']) ? trim($_GET['search']) : '';
?>
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>提货码管理后台</title>
    <style>
        body { font-family: monospace; background: #1e1e1e; color: #d4d4d4; padding: 30px; }
        .header { border-bottom: 2px solid #007acc; padding-bottom: 15px; margin-bottom: 30px; display: flex; justify-content: space-between; }
        .search-box { background: #333; border: 1px solid #555; padding: 10px; color: #fff; width: 250px; }
        .grid { display: grid; grid-template-columns: 1fr 200px; gap: 1px; background: #333; }
        .cell { background: #1e1e1e; padding: 15px; }
        .token-val { color: #f1c40f; font-weight: bold; cursor: pointer; border: 1px dashed #444; padding: 3px 8px; }
    </style>
</head>
<body>

<div class="header">
    <h2>资源提货码查询器</h2>
    <form method="GET">
        <input type="text" name="search" class="search-box" placeholder="搜索包名..." value="<?php echo htmlspecialchars($search); ?>">
        <button type="submit" style="padding:10px; cursor:pointer;">查询</button>
    </form>
</div>

<div class="grid">
    <div class="cell" style="font-weight:bold; color:#007acc;">文件名</div>
    <div class="cell" style="font-weight:bold; color:#007acc;">今日 6 位随机码</div>

    <?php 
    foreach ($files as $file) {
        $fn = basename($file);
        if ($search && stripos($fn, $search) === false) continue;
        $code = get_token($fn, $secret_salt);
        echo "<div class='cell'>$fn</div>";
        echo "<div class='cell'><span class='token-val' onclick='copyIt(\"$code\")'>$code</span></div>";
    }
    ?>
</div>

<script>
function copyIt(text) {
    navigator.clipboard.writeText(text).then(() => alert('已复制: ' + text));
}
</script>

</body>
</html>



```
![image]https://github.com/Coding-01/linux_service_build/blob/main/build_yourls/images/8.png)



# 后台数据
![image]images/9.png)
