## Notes on installing ansible/awx

### First attempt: vanilla installation, no containers (failed)

```sh
sudo apt install postgresql postgresql-contrib
sudo -u postgres createuser --interactive
createdb awx

sudo apt install python-pip software-properties-common

sudo apt-add-repository ppa:ansible/ansible
sudo apt update
sudo apt install ansible

sudo apt install libxml2-dev libxmlsec1-dev libsasl2-dev python-dev libldap2-dev libssl-dev
git clone --depth=1 -b 1.0.1 https://github.com/ansible/awx.git
cd awx/
pip install -r requriements/requirements.txt
pip install -r requirements/requirements_git.txt
./setup.py build
./setup.py install --user

./manage.py collectstatic --clear --noinput
curl -sL https://deb.nodesource.com/setup_6.x | sudo -E bash -
sudo apt install nodejs
npm --unsafe-perm --prefix awx/ui install awx/ui
npm --prefix awx/ui run build-release

cp awx/settings/local_settings.py.example awx/settings/local_settings.py
# Edit accordingly

./manage.py migrate
./manage.py runserver
```

### Second attempt: Docker by the book (success)

Source: https://github.com/ansible/awx/issues/78

```sh
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
curl -sL https://deb.nodesource.com/setup_6.x | sudo -E bash -
sudo apt-add-repository ppa:ansible/ansible
sudo apt update
sudo apt install -y software-properties-common ansible apt-transport-https ca-certificates curl software-properties-common docker-ce nodejs python-pip
pip install docker-py

# You might have to restart the shell after this
sudo usermod -a -G docker $USER

git clone --depth=1 -b 1.0.1 https://github.com/ansible/awx.git
cd awx/installer

# Edit inventory accordingly (comment out dockerhub_base and dockerhub_version if you want to build the images yourself)

ansible-playbook -i inventory install.yml
```
