
### This is an unofficially upgraded installation guide for [Canvas LMS](https://github.com/instructure/canvas-lms), on [Debian 10](https://www.debian.org/News/2019/20190706).  

The QTIMigrationTool needs to be installed to enable copying or importing quiz content.   
Instructions are at https://github.com/instructure/QTIMigrationTool/wiki.  
After installation, ensure the plugin is active in Site Admin -> Plugins -> QTI Converter  
(it should detect the QTIMigrationTool and auto-activate).

## INSTALL  

First check if, python 2.x is installed:  

    root@server:/$  python --version  

The **python-lxml** package is needed:  

    root@server:/$  apt-get install python-lxml  

Then clone the git repo, copy or move files, set the permissions, etc.  

    root@server:/$  cd /var/canvas/vendor
    root@server:/$  git clone https://github.com/instructure/QTIMigrationTool.git QTIMigrationTool
    root@server:/$  chown -R root:root QTIMigrationTool/
    root@server:/$  cd QTIMigrationTool && chmod +x migrate.py
    
Finally: restart the background job manager with:
    
    root@server:/$  /etc/init.d/canvas_init restart

