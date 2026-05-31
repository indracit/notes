
```bash

#ssh login
ssh username@host -p 2229

#secure copy current folder
scp -P 2229 -r username@host:/home/user/dsb/  .

#copy 
cp -rn  ./*  /home/user/dsb/node_modules_2509_bkp/

#ssl verification command
openssl x509 -in cert.crt -noout -text

# supressing pm2 warnings
echo "export NODE_NO_WARNINGS=1" >> ~/.bashrc 
source ~/.bashrc 
pm2 restart all

# key db commands
hget "key" "value"


# connecting to key db
keydb-cli -h host -p 6379 -a password

```


