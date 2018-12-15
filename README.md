# Amazon EMR for Data Science
## Create on-demand Apache Hadoop cluster to analyze Big Data with Spark


##### Easily Run and Scale Apache Spark, Hadoop, HBase, Presto, Hive, and other Big Data Frameworks


![Amazon EMR](https://pbs.twimg.com/media/BwuwVcZIQAA8UgJ.png)

Amazon EMR provides a managed Hadoop framework that makes it easy, fast, and cost-effective to process vast amounts of data across dynamically scalable Amazon EC2 instances. You can also run other popular distributed frameworks such as Apache Spark, HBase, Presto, and Flink in EMR, and interact with data in other AWS data stores such as Amazon S3 and Amazon DynamoDB. EMR Notebooks, based on the popular Jupyter Notebook, provide a development and collaboration environment for ad hoc querying and exploratory analysis.

EMR securely and reliably handles a broad set of big data use cases, including log analysis, web indexing, data transformations (ETL), machine learning, financial analysis, scientific simulation, and bioinformatics.



**EMR benefits**
Since you can set EMR to install Apache Spark, this service is good for for cleaning, reformatting, and analyzing big data. You can use EMR on-demand, meaning you can set it to grab the code and data from a source (e.g. S3 for the code, and S3 or RDS for the data), run the task on the cluster, and store the results somewhere (again s3, RDS, or Redshift) and terminate the cluster.

By using the service in such a way, you can reduce the cost of your cluster significantly. In my opinion, EMR is one of the most useful AWS services for data scientists.

This guide shows how the creation of such EMR cluster for Data Science purposes can be automated by using the AWS CLI.


### Install AWS CLI

Install AWS command line client:

```bash
pip install awscli
```

Configure the AWS command line client:

```bash
aws configure
```

* AWS Access Key ID: \<\<Your public access key\>\>
* AWS Secret Access Key: \<\<Your private access key\>\>
* Default region name: us-east-1
* Default output format: json




### Create the cluster 

The following example CLI command is used to launch a three-node (m4.large) EMR 5.12. cluster with a bootstrap action. The Bootstrap Action will install all the available kernels. It will also install the ggplot and pybrain Python packages and set:

* the Jupyter port to 8885
* the password to jupyter
* the JupyterHub port to 8005

```bash
aws emr create-cluster --release-label emr-5.12.1 \
  --name 'My emr-5.12.1 cluster' \
  --applications Name=Hadoop Name=Hive Name=Spark Name=Pig Name=Tez Name=Ganglia Name=Presto \
  --region us-east-1 \
  --use-default-roles --ec2-attributes KeyName=<your-ec2-key> \ 
  --instance-groups \
    InstanceGroupType=MASTER,InstanceCount=1,InstanceType=m4.large \
    InstanceGroupType=CORE,InstanceCount=2,InstanceType=m4.large \
  --log-uri s3://<your-s3-bucket>/emr-logs/ \
  --bootstrap-actions \
    Name='Install Jupyter notebook'
    Path="s3://aws-bigdata-blog/artifacts/aws-blog-emr-jupyter/install-jupyter-emr5.sh", 
    Args=[--r,--julia,--toree,--torch,--ruby,--ds-packages,--ml-packages,--python-packages,'ggplot nilearn',--port,8885,--password,jupyter,--jupyterhub,--jupyterhub-port,8005,--cached-install,--notebook-dir,s3://<your-s3-bucket>/notebooks/,--copy-samples]
```

Replace `<your-ec2-key>` with your AWS access key and `<your-s3-bucket>` with the S3 bucket where you store notebooks. You can also change the instance types to suit your needs and budget.


Example:

```bash
aws emr create-cluster --release-label emr-5.12.1 \
  --name 'My emr-5.12.1 cluster' \
  --applications Name=Hadoop Name=Hive Name=Spark Name=Pig Name=Tez Name=Ganglia Name=Presto \
  --region us-east-1 \
  --use-default-roles --ec2-attributes KeyName=cluster_keypair \ 
  --instance-groups \
    InstanceGroupType=MASTER,InstanceCount=1,InstanceType=m4.large \
    InstanceGroupType=CORE,InstanceCount=2,InstanceType=m4.large \
  --log-uri s3://achilleaskn-emr-data-science/emr-logs/ \
  --bootstrap-actions \
    Name='Install Jupyter notebook',
    Path="s3://aws-bigdata-blog/artifacts/aws-blog-emr-jupyter/install-jupyter-emr5.sh", 
    Args=[--r,--julia,--toree,--torch,--ruby,--ds-packages,--ml-packages,--python-packages,'ggplot nilearn',--port,8885,--password, mystrongjupyter,--jupyterhub,--jupyterhub-port,8005,--cached-install,--notebook-dir,s3://achilleaskn-emr-data-science/notebooks/,--copy-samples]
```

A cluster-id should be returned, as below:

```bash
{
    "ClusterId": "j-XXXXXXXXXXXXXXX"
}
```



### Get information about the cluster

##### Cluster details

```bash
aws emr describe-cluster --cluster-id j-XXXXXXXXXXXXXXX
```

##### Cluster state

```bash
aws emr describe-cluster --cluster-id j-XXXXXXXXXXXXXXX | grep \"State\"
```

What are different cluster states?

State     | Description
:---------|:--------- 
 STARTING |The cluster provisions, starts, and configures EC2 instances.
 BOOTSTRAPPING | Bootstrap actions are being executed on the cluster. 
 RUNNING | A step for the cluster is currently being run. 
 WAITING | The cluster is currently active, but has no steps to run.
 TERMINATING | The cluster is in the process of shutting down.
 TERMINATED | The cluster was shut down without error.
 TERMINATED_WITH_ERRORS | The cluster was shut down with errors.

<br/>

##### Master node public DNS:

```bash
aws emr describe-cluster --cluster-id j-XXXXXXXXXXXXXXX | grep \"MasterPublicDnsName\"
```


You should get somthing like:
```
ec2-###-##-##-###.compute-1.amazonaws.com"
```




### Add rules to access the cluster

#### Adding Rules to Your Security Group
<!-- To access Jup......ssh you have to


set up the SSH tunnel and web proxy.

https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-ssh-tunnel.html -->


##### Get Master's security group

```bash
aws emr describe-cluster --cluster-id j-XXXXXXXXXXXXXXX | grep \"EmrManagedMasterSecurityGroup\"
```

It should return the id of the SecurityGroup that is applied on the Master:

```
"EmrManagedMasterSecurityGroup": "sg-0ea4d4bxxx8fe235e"
```


#### To add a rule that allows inbound SSH traffic


The following command adds a rule to enable inbound traffic on TCP port 22 (SSH) to the security group with the ID sg-903004f8::

```bash
aws ec2 authorize-security-group-ingress --group-id sg-XXXXXXXXXXX --protocol tcp --port 22 --cidr 0.0.0.0/0
```
Replcase sg-XXXXXXXXXXX with Master's Security Group id described above.


#### To add a rule that allows inbound Jupyter & JupyterHub traffic

For Jupyter:
```bash
aws ec2 authorize-security-group-ingress --group-id sg-XXXXXXXXXXX --protocol tcp --port 8885 --cidr 0.0.0.0/0
```
For JupyterHub
```bash
aws ec2 authorize-security-group-ingress --group-id sg-XXXXXXXXXXX --protocol tcp --port 8005 --cidr 0.0.0.0/0
```
Replcase sg-XXXXXXXXXXX with the Master's Security Group id.




### Connect to the cluster via ssh

You can also conect to the master node via ssh with the following command:

```bash
aws emr ssh --cluster-id -XXXXXXXXXXXXXXX --key-pair-file <your-ec2-key>
```

Replace the `<your-ec2-key>` with the path/file of the key.





### Terminate the cluster

When you are done, remember to terminate the cluster!

```bash
aws emr terminate-clusters --cluster-id j-XXXXXXXXXXXXXXX
```

...and confirm that it is terminating:

```bash
aws emr describe-cluster --cluster-id j-XXXXXXXXXXXXXXX | grep \"State\"
```

You should see:

```bash
    "State": "TERMINATING"
        "State": "TERMINATING"
        "State": "TERMINATING"
```


## References

http://people.duke.edu/~ccc14/sta-663-2016/21H_Spark_Cloud.html

https://aws.amazon.com/blogs/big-data/running-jupyter-notebook-and-jupyterhub-on-amazon-emr/