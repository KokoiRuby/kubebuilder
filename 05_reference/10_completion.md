```bash
$ kubebuilder completion [bash|fish|powershell|zsh]
```

```bash
sudo apt-get update
sudo apt-get install bash-completion

# zsh
$ echo “/usr/local/bin/zsh” > /etc/shells
$ chsh -s /usr/local/bin/bash

# ++ to ~/.zshrc
# kubebuilder autocompletion
if [ -f /usr/local/share/bash-completion/bash_completion ]; then
. /usr/local/share/bash-completion/bash_completion
fi
. <(kubebuilder completion bash)
```

