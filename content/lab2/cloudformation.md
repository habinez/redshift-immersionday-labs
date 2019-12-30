---
title: "Cloudformation"
weight: 22
---

## Cloud Formation
To *skip this lab* and complete the loading of this sample data into an **existing cluster** using cloud formation, use the following link.

[![Launch](/images/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?#/stacks/new?stackName=ImmersionLab2&templateURL=https://s3-us-west-2.amazonaws.com/redshift-immersionday-labs/lab2.yaml)

[lab2.yaml](lab2.yaml)

To *skip this lab* and complete the loading of this sample data into a **new cluster** using cloud formation, use the following link.

[![Launch](/images/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?#/stacks/new?stackName=ImmersionLab2&templateURL=https://s3-us-west-2.amazonaws.com/redshift-immersionday-labs/lab2%2Bcluster.yaml)

Note: These cloud formation templates will create a Lambda function which will trigger an asynchronous Glue Python Shell script.  The Glue script assumes that the Redshift Cluster is publically accessible with an ingress rule of 0.0.0.0/0.  To monitor the load process and diagnose any load errors, see the following Cloudwatch Logs stream:
```
https://console.aws.amazon.com/cloudwatch/home?#logStream:group=/aws-glue/python-jobs/output
```

