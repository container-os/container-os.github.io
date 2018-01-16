# Contribute to Container-OS!

*We welcome all contributions.*

Fork the repos from github. https://github.com/container-os/

Please use github pull request to send us your patch.
Or use git send-email to send your patch.

#### ~/.gitconfig
```
[includeIf "gitdir:~/github/container-os.github.io/"]
        path = ~/.git/cos.gitconfig
```

#### ~/.git/cos.gitconfig
```
[sendemail]
        #smtpEncryption = tls
        smtpServer = smtp.xxx.com
        smtpUser = hello@xxx.com
        smtpServerPort = 25
        smtppass = hello_password
        confirm = auto
        to = system-engineering@easystack.cn
```

Chat with the developers on IRC: freenode.org, channel #[#container-os](https://riot.im/app/#/room/#freenode_#container-os:matrix.org)

