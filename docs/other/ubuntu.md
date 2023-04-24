# Ubuntu

> 由于新公司采用乌班图作为服务器，这边记录一些日常使用到的东西

```shell
# 生成密钥，中间会让你输密码和地址
ssh-keygen 

#1.切换ssh登录方式
#现在，在 root目录中生成了一个 .ssh 的隐藏目录，内含两个密钥文件。id_rsa 为私钥，id_rsa.pub 为公钥
cd /root/.ssh  #进入秘钥所在 的目录
cat id_rsa.pub >> authorized_keys #将公钥转换成认证用（保证旧的不存在，否则会两个共用）
chmod 600 authorized_keys 	#修改权限满足认证要求
chmod 700 ~/.ssh			#修改文件夹权限满足认证要求
#编辑 /etc/ssh/sshd_config 文件
#确保秘钥认证的2个配置是yes状态
RSAAuthentication yes
PubkeyAuthentication yes
#关闭密码登录
PasswordAuthentication no
#重启SSH服务，至此完成
sudo  service sshd restart

#2.注意新版的生成的密钥前缀为OPENSSH会带来各种问题，如果要使用RSA的可以用
#这个命令，当重新生成后重复上述操作即可，但是它会让你覆盖文件实际上是追加，并不会导致之前的密钥失效
ssh-keygen -m PEM -t rsa -b 4096


```

