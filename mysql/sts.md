

### Master-Sts

```
[root@kube03 sts]# cat 2master-sts.json


{
    "apiVersion": "apps/v1",
    "kind": "StatefulSet",
    "metadata": {
        "labels": {
            "name": "mysql-master"
        },
        "name": "mysql-master",
        "namespace": "yulibaozi"
    },
    "spec": {
        "podManagementPolicy": "OrderedReady",
        "replicas": 1,
        "revisionHistoryLimit": 10,
        "selector": {
            "matchLabels": {
                "app": "mysql-master"
            }
        },
        "serviceName": "mysql-master",
        "template": {
            "metadata": {
                "labels": {
                    "app": "mysql-master"
                }
            },
            "spec": {
                "containers": [
                    {
                        "env": [
                            {
                                "name": "MYSQL_ROOT_PASSWORD",
                                "value": "root"
                            },
                            {
                                "name": "MYSQL_PASSWORD",
                                "value": "root"
                            },
                            {
                                "name": "MYSQL_REPLICATION_USER",
                                "value": "repl"
                            },
                            {
                                "name": "MYSQL_REPLICATION_PASSWORD",
                                "value": "repl"
                            }
                        ],
                        "image": "yulibaozi/mysql-master:201807261706",
                        "imagePullPolicy": "IfNotPresent",
                        "name": "mysql-master",
                        "ports": [
                            {
                                "containerPort": 3306,
                                "protocol": "TCP"
                            }
                        ],
                        "resources": {},
                        "terminationMessagePath": "/dev/termination-log",
                        "terminationMessagePolicy": "File",
                        "volumeMounts": [
                            {
            ③                    "mountPath": "/var/lib/mysql",
                                "name": "data"
                            }
                        ]
                    }
                ],
                "dnsPolicy": "ClusterFirst",
                "restartPolicy": "Always",
                "schedulerName": "default-scheduler",
                "securityContext": {},
                "terminationGracePeriodSeconds": 10,
                "volumes": [
                    {
                        "name": "data",
                        "rbd": {
                            "fsType": "xfs",
            ②               "image": "k8s_image01",
                            "keyring": "/etc/ceph/keyring",
                            "monitors": [
            ⑤                   "192.168.0.199:6789"
                            ],
                            "pool": "rbd",
                            "secretRef": {
            ①                   "name": "ceph-secret"
                            },
            ④               "user": "admin"
                        }
                    }
                ]
            }
        },
        "updateStrategy": {
            "rollingUpdate": {
                "partition": 0
            },
            "type": "RollingUpdate"
        }
    },
    "status": {
    }
}

相关解释：
①: secret的名字,就是上文中创建的secret名字
②：ceph管理员给你的块名字
③：容器中挂载的目录,这么目录就是mysql存数据的目录,如果不知道怎么做就不要改
④：ceph的用户名，问下ceph管理员是什么，如果不是admin就得改
⑤：monitors:ceph的monitor地址，找ceph管理员要，可以是一个,也可以是多个
```

### Slave-Sts
```
[root@kube03 sts]# cat 3slave-sts.json

{
    "apiVersion": "apps/v1",
    "kind": "StatefulSet",
    "metadata": {
        "labels": {
            "name": "mysql-slave"
        },
        "name": "mysql-slave",
        "namespace": "yulibaozi"
    },
    "spec": {
        "podManagementPolicy": "OrderedReady",
        "replicas": 1,
        "revisionHistoryLimit": 10,
        "selector": {
            "matchLabels": {
                "name": "mysql-slave"
            }
        },
        "serviceName": "mysql-slave",
        "template": {
            "metadata": {
                "creationTimestamp": null,
                "labels": {
                    "name": "mysql-slave"
                }
            },
            "spec": {
                "containers": [
                    {
                        "env": [
                            {
                                "name": "MYSQL_ROOT_PASSWORD",
                                "value": "root"
                            },
                            {
                                "name": "MYSQL_PASSWORD",
                                "value": "root"
                            },
                            {
                                "name": "MYSQL_REPLICATION_USER",
                                "value": "repl"
                            },
                            {
                                "name": "MYSQL_REPLICATION_PASSWORD",
                                "value": "repl"
                            },
                            {
                                "name": "MASTER_HOST",
        ①                        "value": "mysql-slave.yulibaozi"
                            }
                        ],
                        "image": "yulibaozi/mysql-slave:201807261706",
                        "imagePullPolicy": "IfNotPresent",
                        "name": "mysql-slave",
                        "ports": [
                            {
                                "containerPort": 3306,
                                "protocol": "TCP"
                            }
                        ],
                        "resources": {},
                        "terminationMessagePath": "/dev/termination-log",
                        "terminationMessagePolicy": "File",
                        "volumeMounts": [
                            {
        ②                       "mountPath": "/var/lib/mysql",
                                "name": "data"
                            }
                        ]
                    }
                ],
                "dnsPolicy": "ClusterFirst",
                "restartPolicy": "Always",
                "schedulerName": "default-scheduler",
                "securityContext": {},
                "terminationGracePeriodSeconds": 10,
                "volumes": [
                    {
                        "name": "data",
                        "rbd": {
                            "fsType": "xfs",
        ③                   "image": "k8s_image02",
                            "keyring": "/etc/ceph/keyring",
        ⑥                   "monitors": [
                                "192.168.0.199:6789"
                            ],
                            "pool": "rbd",
                            "secretRef": {
        ④                       "name": "ceph-secret"
                            },
                            "user": "admin"
        ⑤               }
                    }
                ]
            }
        },
        "updateStrategy": {
            "rollingUpdate": {
                "partition": 0
            },
            "type": "RollingUpdate"
        }
    },
    "status": {
    }
}

相关解释：
①：如果你的namespace不是yulubaozi,需要把yulibaozi替换成你的命名空间名字
②：需要配置mysql的数据目录，可以选择不修改，默认是它
③：ceph管理分配给你的rbdImage名字，注意:不得与master是同一个名字（同一个名字是指:同一个key，同一个rbdImage名字）
④：secret的名字，secret里面的Key和rbdImage相对应，我这里master和slave使用的相同的key，而rbdImage名字不相同，所以master可以和slave使用一样的secret；如果你的key和rbdImage是不同的两套，那么需要创建两个secret,master一个,slave一个,secret的创建方式在上面提到了。
⑥：monitors:ceph的monitor地址，找ceph管理员要，可以是一个,也可以是多个,可以和master一致也可和master不一致
```