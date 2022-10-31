# Ubuntu

> 由于新公司采用乌班图作为服务器，这边记录一些日常使用到的东西

```shell
#1.切换ssh登录方式
#现在，在 user 用户的home目录中生成了一个 .ssh 的隐藏目录，内含两个密钥文件。id_rsa 为私钥，id_rsa.pub 为公钥
ssh-keygen 
cd /home/user/.ssh  #进入秘钥所在 的目录
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

#2.

```

