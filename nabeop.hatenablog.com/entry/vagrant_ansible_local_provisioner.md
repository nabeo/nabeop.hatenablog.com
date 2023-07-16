---
Title: 最近は Vagrant では ansible_local でプロビジョニングしている
Category:
- vagrant
- ansible
Date: 2019-06-08T16:31:46+09:00
URL: https://nabeop.hatenablog.com/entry/vagrant_ansible_local_provisioner
EditURL: https://blog.hatena.ne.jp/nabeop/nabeop.hatenablog.com/atom/entry/17680117127190825871
---

vagrant で手元の VirtualBox に検証環境を作る場合は [Ansible Local Provisioner](https://www.vagrantup.com/docs/provisioning/ansible_local.html) を使っています。

昔は [Ansible Provisioner](https://www.vagrantup.com/docs/provisioning/ansible.html) を使っていたけど、手元環境を汚さずに仮想ホスト上で ansible を直接実行できることにメリットを感じています。

Vagrantfile はこんな感じで準備しておきます

```ruby
Vagrant.configure(2) do |config|
  config.vm.synced_folder './ansible', '/home/vagrant/ansible',
                          create: true, owner: 'vagrant', group: 'vagrant'

  config.vm.define 'vm01' do |node|
    node.vm.provider 'virtualbox' do |vb|
      vb.cpus = 1
      vb.memory = 512
      vb.gui = false
    end
    node.vm.hostname = 'vm01'
    node.vm.box = 'ubuntu/cosmic64'
    node.vm.provision :ansible_local do |ansible|
      ansible.compatibility_mode = '2.0'
      ansible.playbook = '/home/vagrant/ansible/all.yml'
      ansible.inventory_path = '/home/vagrant/ansible/inventories/hosts'
      ansible.limit = 'clients'
    end
  end
```

`Vagrantfile` があるディクトリに `ansible` ディレクトリを作って、その下に ansible playbook を配置しています。で、`asible` ディレクトリは `config.vm.synced_folder './ansible', '/home/vagrant/ansible'` で VM 上の `/home/vagrant/ansible` として配置しています。

Ansible Local Provisioner を使うときの注意点としては ansible は vm 上で動くため、inventory file は自分自身をプロビジョニング対象にする必要があります。vagrant-hosts プラグインなどを使って hosts ファイルを管理してもいいのですが、僕は inventory file を ansible の role ごとに分けて管理しています。したがって、 `ansible` ディレクトリの中身はこんな感じになっています

```
ansible
├── all.yml <--- ansible.playbook で指定している
├── clients.yml
|
├── inventories
│   └── clients <--- ansible.inventory_path で指定している
|
└── roles
    ├── clients
    └── role_template

```

また、inventory file は ansible をローカル環境を対象に実行する必要があるので、以下のように書いています。

```
[clients]
127.0.0.1 ansible_connection=local
```
