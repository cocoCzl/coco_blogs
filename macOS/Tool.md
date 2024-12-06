## Homebrews

错误信息

1. curl: (7) Failed to connect to raw.githubusercontent.com port 443: Connection refused......  
   可能原因：github 的一些域名的DNS解析被污染，导致DNS
   解析过程无法通过域名取得正确的IP地址。可以通过修改/etc/hosts文件可解决该问题。  
   打开网站: https://www.ipaddress.com/
   查询一下 raw.githubusercontent.com对应的IP地址  
   使用vim /etc/hosts命令打开不能访问的机器的hosts文件，添加如下内容：  
   185.199.108.133 raw.githubusercontent.com  
   185.199.109.133 raw.githubusercontent.com  
   185.199.110.133 raw.githubusercontent.com  
   185.199.111.133 raw.githubusercontent.com
2. error: RPC failed; curl 56 Recv failure: Connection reset by peer或者curl 92 HTTP/2 stream 5 was not closed cleanly:
   CANCEL (err 8)  
   可以尝试 增大 Git 缓冲区可以帮助解决传输中断问题  
   git config --global http.postBuffer 524288000  
   增加 Git 的超时时间来防止网络不稳定引起的中断  
   git config --global http.lowSpeedLimit 0  
   git config --global http.lowSpeedTime 999999
3. 错误：无法下载 https://formulae.brew.sh/api/formula.jws.json！  
   多执行几次安装命令
4. Warning: /opt/homebrew/bin is not in your PATH  
   添加 Homebrew 到 PATH  
   vim ~/.zshrc  
   在末尾添加：export PATH="/opt/homebrew/bin:$PATH"  
   重新加载配置文件：source ~/.zshrc