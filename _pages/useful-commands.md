---
permalink:          /comandos-uteis/
permalink_en-US:    /useful-commands/
namespace:          useful-commands
author_profile:     true
---
{% translate_file _pages/useful-commands.md %}

## File Transfer

### Send/Receive Files using NC

More info at [Using Netcat for File Transfers (nakkaya.com)](https://nakkaya.com/2009/04/15/using-netcat-for-file-transfers/)

```bash
# Receiving Machine
nc -l -p 1234 > out.file

# Sending Machine
nc -w 3 [destination] 1234 < out.file
```

## Upgrading Shell

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
<Ctrl+Z>
stty raw -echo; fg
export TERM=screen-256color; stty rows 50 columns 200
```

