# Ranger Atlas (Hortonia Bank)

## Demo overview

Demo overview can be found [here](https://community.hortonworks.com/articles/151939/hdp-securitygovernance-demo-kit.html) 

## Versions tested

Tested with:
- [x] HDP 2.6.5 / Ambari 2.6.2
- [x] HDP 2.6.4 / Ambari 2.6.1
- [x] HDP 2.6.3 / Ambari 2.6.0
- [x] HDP 2.6.4 Sandbox

## Option #1: Fresh install of HDP plus demo

- Pre-reqs:
  - Launch a single vanilla Centos/RHEL 7.x VM (e.g. on local VM or openstack or cloud provider of choice) 
  - The VM should not already have any Ambari or HDP components installed (e.g. do NOT run script on HDP sandbox)
  - The VM requires 4 vcpus and ~17-18 GB RAM once all services are running and you execute a query, so m3.2xlarge size is recommended
  
- Login as root and run setup.sh as below:
```
sudo su
cd
export stack=cl1
export ambari_pass=admin
export ambari_stack_version=2.6
export host_count=5
export enable_kerberos=false
export ambari_version="2.6.1.5-3" 
export zeppelin_host=test-cluster-7.novalocal
export atlas_host=test-cluster-9.novalocal
export ranger_host=test-cluster-10.novalocal
export hive_host=test-cluster-6.novalocal
export hcat_host=test-cluster-6.novalocal
export kafka_broker=test-cluster-7.novalocal
export zookeeper_host=test-cluster-6.novalocal
curl -sSL https://raw.githubusercontent.com/asirna/masterclass/master/ranger-atlas/setup.sh | sudo -E bash  
```

- This will run for about 30min. Once complete, proceed to part 2 below and complete the manual steps to enable Hive LLAP

- Note: if you will later be registering this cluster with DSS, you can skip the manual steps to enable LLAP (as the profiler will not be able to launch any jobs on the default YARN queue after LLAP is enabled)

## Option #2: Setup demo on HDP Sandbox 

- Pre-reqs (refer to [sandbox guide](https://hortonworks.com/tutorial/sandbox-deployment-and-install-guide) for detailed steps)
  - Download HDP 2.6.4 Sandbox from [here](http://hortonworks.com/sandbox) 
  - Allocate 4 vcpus and 12+ GB RAM 
  - Start Sandbox
  - Login via SSH as root/hadoop (you will be required to change the passord)
  - Run `ambari-admin-password-reset` to reset the Ambari admin password to one of your choosing e.g. BadPass#1

  
- Connect via SSH to sandbox as root, set your Ambari admin password and run setup_sandbox.sh (note this requires VM has access to internet):
```
export ambari_pass=BadPass#1
curl -sSL https://raw.githubusercontent.com/asirna/masterclass/master/ranger-atlas/setup_sandbox.sh | sudo -E sh
```


- This will run for about 10min. Once complete, proceed to part 2 below and complete the manual steps

- Note: enabling LLAP (along with all the other components required) may require extra memory resources on your VM


## Login details 

- Ambari port: 8080 login: admin/BadPass#1
- Ranger port: 6080 login: admin/admin
- Atlas port: 21000 login: admin/admin
- Zeppelin port: 9995 login: ivanna_eu_hr/BadPass#1 OR joe_analyst/BadPass#1 OR etl_user/BadPass#1

## Manual steps

- Once script is complete, there are manual steps required to complete the demo setup 
- [X] The previous steps of manually creating a tag service in Ranger, importing tag policies and associating it with Hive/Hbase/Kafka services are no longer required. These are now taken care of by the script

- [ ] In order to be able to run consent related queries, the below additional pre-reqs must be taken care of:
  - [ ] From Ambari: enable/start Interactive Hive (the consent queries require Hive 2.x+)
  - [ ] From Ambari: restart Zeppelin (to make it aware of %jdbc(hive_interactive))
  - [ ] From Ranger: enable Hive row filter policy called 'filter: ww_customers for consent' (disable existing policy first)

- [ ] Login to Zeppelin as end users (ivanna_eu_hr and joe_analyst and etl_user) and run through demo Hive queries one by one in the prebuilt notebooks. After running each query, refresh Ranger audits page to understand what transpired

  ## Demo walkthrough
  
  - Detailed walkthrough of demo steps available [here](https://community.hortonworks.com/articles/151939/hdp-securitygovernance-demo-kit.html)

  ## Other things to try
- Simulate users trying to randomly access Hive tables to generate more interesting audits
```
/tmp/masterclass/ranger-atlas/HortoniaMunichSetup/audit_simulator.sh
```

- Install Ranger Audits Banana dashboard to visuaize audits


  ## How does it work?
- The script basically:
  - uses [Ambari bootstrap](https://github.com/seanorama/ambari-bootstrap) to install Ambari, generate a blueprint and deploy HDP cluster that includes Ranger/Atlas
  - uses Ranger APIs to import service defs, create tag repo and import policies for HDFS/Hive/Hbase/Kafka
  - import tags into Atlas
  - imports sample Hive data (which also creates HDFS/Hive entities in Atlas)
  - [uses Atlas APIs to associate tags with Hive/Kafka/Hbase/HDFS entities](https://community.hortonworks.com/articles/189615/atlas-how-to-automate-associating-tagsclassificati.html)
  - enables kerberos


  ## Troubleshooting

- While running script, beeline stuck at analyzing tables?
  - Check the YARN UI on port 8088: there is likely one job in RUNNING and one in ACCEPTED state. Kinit and kill the one in RUNNING state:
```
kinit -kVt /etc/security/keytabs/rm.service.keytab rm/$(hostname -f)@HWX.COM
yarn application -kill <application_id>
```

- Ranger audits not picking up tags?
  - Restart Ranger tagsync process via Ambari and re-try

- Interactive Hive/LLAP not starting? 
  - Ambari shows `The cluster is not started yet (InvalidACL); will retry`
  - Regenrate keytabs via Ambari and restart services. See [here](https://community.hortonworks.com/articles/125751/iop-v-425-to-hdp-v-26x-hsi-start-fails-with-error.html) for more info
  
- Zeppelin complains of `Prefix not found.`?
  - Zeppelin service was not restarted after enabled Interactive Hive. Restart Zeppelin via Ambari

