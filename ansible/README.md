# Run playbooks locally

ansible-playbook playbooks/site.yml
ansible-playbook playbooks/site.yml --tags="postinstall,configuration" --limit valhalla --check -vvv
ansible-playbook playbooks/site.yml --limit 'all:!valhalla' --check -vvv
ansible-playbook playbooks/site.yml --limit 'all:!valhalla' --check -vvv
ansible-playbook playbooks/site.yml --limit 0 --check -vvv
