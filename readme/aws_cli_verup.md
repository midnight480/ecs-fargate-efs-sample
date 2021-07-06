
# Precheck

```
aws --version
```

# Install AWS CLI v2

```
sudo yum update -y
sudo pip uninstall awscli -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
export PATH=$PATH:/usr/local/bin/
complete -C aws_completer aws
echo '# aws completer' >> ~/.bash_profile
echo complete -C \'$(which aws_completer)\' aws >> ~/.bash_profile
source ~/.bash_profile
```
