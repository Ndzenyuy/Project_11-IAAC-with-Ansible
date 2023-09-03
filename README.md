# Project 11: Infrastructure as Code with Ansible

Greetings fam, ever wondered about writting code and lauching an infrastructure without having to open the different services on the console and create and wait for the creation to finish before moving on to the next service? Well Ansible brings us good news. With Ansible Playbooks, I was able to build an Infrastructure from a Control machine(EC2) instance. The code is stored in my repo on Github(edited on my local machine), then the Control machine, with an Ansible installation clones the code, from it, builds the infrastruceture. I actually used a variable file, which was imported into the playbook, the vars_file serves the purpose of making infrastructure code reusable, say launch the same infrastructure in another region, by just modifying the region's specs say the ami etc). One key lession here never never neveeeeer to be forgotten: use IAM Roles for the control machine rather than hardcoding your Secret keys into the playbook, for it can be mistakenly pushed to Github, by the way if you push your secret keys to Github, you will be detained in DevOps top security detaintion center for 3months(i hope you know am joking. As a teacher, i used to associate an important concept to a joke, the students will pause for a while and laugh, the concept will stick since they'll remember the joke)

## Architecture
![](architecture)

