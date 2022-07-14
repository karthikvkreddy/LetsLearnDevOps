### About the project:
- Re-Architect services for AWS Cloud
- Artitecture to boost agility or improve business continuity

### Scenario:
- Projects services running on phy/virtual/cloud machines.
- varieties of services that powers your project runtime.
- We need cloud computing team, monitoring, DC OPS team etc involved to manage workload.

### Problems:
- Opertaional overhead
- strugguling  with uptime & scaling
- upfront CapEx & regular OpEx
- Manual Process / Difficult to automate.

### Solution:
- PAAS & SAAS 
- IAAC
- Flexibility
- scaling
- Ease of infra managing

### AWS Services
- BeanStack
- S3/EFS
- RDS
- ELASTIC Cache
- Active MQ
- Route 53
- Cloudfront- CDN

### Objectives:
- Flexible infra
- IAAC
- PAAS
- SAAS
- NO Upfront cost

### Architecture:
---
![alt text](..\images\3-arc.png)


## Flow of Execution:
1. Create pair for beanstalk instance login
2. create Security group for elasticcache. RDS & active MQ
3. Create
    - RDS
    - amazon elastic cache
    - amazon active mq
4. create elastic beanstalk env
5. update SG of backend to allow traffic from bean SG
6. Update SQ of backend to allow internet traffic
7. Launch EC2 instances for DB initializating
8. Login to the instance and initiliaze RDS DB
9. Change healthchek on beanstalk to /login
10. Add 443 https listner to ELB
11. Build Artifact with backend info
12. Deploy Artifact to beanstalk
13. Create CDN with SSL Cert
14. Update Entry in GoDaddt DNS Zones
15. Test the url