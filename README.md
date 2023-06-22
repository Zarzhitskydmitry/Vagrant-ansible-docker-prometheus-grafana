Telegram-bot-python
===================

Make install python3 to linux RHEL(CentOS)

    $ wget https://www.python.org/ftp/python/3.9.10/Python-3.9.10.tar.x
    $ yum install xz -y
    $ tar -xpJf Python-3.9.10.tar.xz
    $ cd Python-3.9.10
    $ yum groupinstall "Development tools" -y
    $ yum install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel -y
    $ ./configure --enable-optimizations
    $ make install
    $ ln -s /usr/local/bin/python3 /usr/bin/python3


Installing
==========

You can install or upgrade ``python-telegram-bot`` via

.. code:: shell

    $ pip3 install python-telegram-bot --upgrade

To install a pre-release, use the ``--pre`` `flag <https://pip.pypa.io/en/stable/cli/pip_install/#cmdoption-pre>`_ in addition.

You can also install ``python-telegram-bot`` from source, though this is usually not necessary.

.. code:: shell

    $ git clone https://github.com/python-telegram-bot/python-telegram-bot
    $ cd python-telegram-bot
    $ python setup.py install
    
 Now download the script itself for execution
    
    $ git clone https://github.com/Zarzhitskydmitry/Telegram-bot-asterisk
    $ cd Telegram-bot-asterisk/
    
Set script execution privileges

    $ chmod +x ./bot.sh ./apache_status.sh ./aster_trunk.sh ./sip_show_registry.sh ./botsipr.sh ./cdr-clear.sh
    
In the bot file.py I filled in all three blocks, according to other functions. To make the bot work, you need to run the file bot.sh . For convenience, we will create a separate service for the Telegram bot. Create the necessary file and set the rights:

    $ touch /etc/systemd/system/telegram-bot.service
    $ chmod 664 /etc/systemd/system/telegram-bot.service

Then go to the service file:
    
    $ nano /etc/systemd/system/telegram-bot.service
    
 ```bash
    [Unit]
    Description=Telegram bot
    After=network.target
    [Service]
    ExecStart=/'YOUR DIRECTORY'/Telegram-bot-asterisk/bot.sh
    [Install]
    WantedBy=multi-user.target
  ```
Now we add the service to the startup and run:

    $ systemctl start telegram-bot.service
    $ systemctl enable telegram-bot.service
    $ systemctl status telegram-bot.service

Create symbolic link 

    $ ln -s ./apache_status.sh /usr/local/sbin/apachestatus
    $ ln -s ./aster_trunk.sh /usr/local/sbin/astertrunk
    $ ln -s ./botsipsr.sh /usr/local/sbin/botsipsr
    $ ln -s ./botsipr.sh /usr/local/sbin/sip_r
    $ ln -s ./sip_show_registry.sh /usr/local/sbin/sip_show_registry

Quick Start
===========

Our Wiki contains an `Introduction to the API <https://github.com/python-telegram-bot/python-telegram-bot/wiki/Introduction-to-the-API>`_ explaining how the pure Bot API can be accessed via ``python-telegram-bot``.
Moreover, the `Tutorial: Your first Bot <https://github.com/python-telegram-bot/python-telegram-bot/wiki/Extensions-%E2%80%93-Your-first-Bot>`_ gives an introduction on how chatbots can be easily programmed with the help of the ``telegram.ext`` module.

Resources
=========

- The `package documentation <https://docs.python-telegram-bot.org/>`_ is the technical reference for ``python-telegram-bot``.
  It contains descriptions of all available classes, modules, methods and arguments as well as the `changelog <https://docs.python-telegram-bot.org/changelog.html>`_.
- The `wiki <https://github.com/python-telegram-bot/python-telegram-bot/wiki/>`_ is home to number of more elaborate introductions of the different features of ``python-telegram-bot`` and other useful resources that go beyond the technical documentation.
- Our `examples section <https://docs.python-telegram-bot.org/examples.html>`_ contains several examples that showcase the different features of both the Bot API and ``python-telegram-bot``.
  Even if it is not your approach for learning, please take a look at ``echobot.py``. It is the de facto base for most of the bots out there.
  The code for these examples is released to the public domain, so you can start by grabbing the code and building on top of it.
- The `official Telegram Bot API documentation <https://core.telegram.org/bots/api>`_ is of course always worth a read.
