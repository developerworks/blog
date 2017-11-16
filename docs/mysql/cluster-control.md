wget http://repo.severalnines.com/severalnines-repos.asc -O- | sudo apt-key add -
wget http://www.severalnines.com/downloads/cmon/install-cc
chmod +x install-cc
INNODB_BUFFER_POOL_SIZE=256 ./install-cc
