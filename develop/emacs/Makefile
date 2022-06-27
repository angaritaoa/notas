install :
	sudo dnf install -y emacs git ripgrep fd-find ShellCheck
	rm -rf ~/.emacs.d
	ln -fns $(shell pwd)/doom.d ~/.doom.d
	git clone https://github.com/doomemacs/doomemacs ~/.emacs.d
	~/.emacs.d/bin/doom install
