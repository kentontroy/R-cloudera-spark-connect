# cloudera-rstudio-spark-connect

## Create a CDE session of type 'spark-connect' if necessary
```
% cat cde_create_session.sh
#!/bin/bash

./cde session create \
  --name test-spark-connect \
  --type spark-connect \
  --description "Please do not delete - in use by Posit" \
  --ttl 8h

% ./cde_create_session.sh
```

## Setup pre-requisistes for C++ bindings
```
% cat /etc/redhat-release
CentOS Linux release 7.9.2009 (Core)

% sudo yum group install "Development Tools"
% cat $HOME/.R/Makevars
CXX11 = g++
CXX14 = g++
CXX17 = g++
CXX17STD = -std=c++17 -fPIC

% sudo yum install devtoolset-9-toolchain
% gcc --version
gcc (GCC) 9.3.1 20200408 (Red Hat 9.3.1-2)

```
## Instantiate python virtual environment
```
% cd /home/centos/posit
% scl enable devtoolset-9 bash
% . cdeconnect/bin/activate

% export GRPC_DEFAULT_SSL_ROOTS_FILE_PATH=/home/centos/posit/cdeconnect/lib/python3.8/site-packages/certifi/cacert.pem

% cat cdeconnect/lib/python3.8/site-packages/certifi/cacert.pem
# ECS Master
-----BEGIN CERTIFICATE-----
MIIDKTCCAs+gAwIBAgIQKBlOEAQfW+WR42MQDghqazAKBggqhkjOPQQDAjAkMSIw
......
# xlsxlrpb.cde-558hm9hd.apps.cdppvcds.com
-----BEGIN CERTIFICATE-----
MIIDMTCCAhmgAwIBAgIUI080GcfjKeCW8Iq9Ex6rSnVK3q8wDQYJKoZIhvcNAQEL
......
```

## Execute R code for package installation and setup
```
% R
> install.packages("reticulate")
> install.packages("remotes")
> remotes::install_github("mlverse/pysparklyr", ref = "cde")

> library(sparklyr)
> library(pysparklyr)
> library(remotes)
> library(reticulate)

> reticulate::use_virtualenv("/home/centos/posit/cdeconnect", required=TRUE)
> reticulate::py_list_packages()
                    package     version                      requirement
1                cdeconnect       0.1.0                cdeconnect==0.1.0
2                   certifi    2024.2.2                certifi==2024.2.2
3                      cffi      1.16.0                     cffi==1.16.0
4        charset-normalizer       3.3.2        charset-normalizer==3.3.2
5            ConfigArgParse         1.7              ConfigArgParse==1.7
6              configparser       6.0.1              configparser==6.0.1
7              cryptography      42.0.5             cryptography==42.0.5
8               distconfig3       1.0.1               distconfig3==1.0.1
9  googleapis-common-protos      1.63.0 googleapis-common-protos==1.63.0
10                   grpcio      1.62.1                   grpcio==1.62.1
11            grpcio-status      1.62.1            grpcio-status==1.62.1
12                     idna         3.6                        idna==3.6
13              load-dotenv       0.1.0               load-dotenv==0.1.0
14                    numpy      1.24.4                    numpy==1.24.4
15                   pandas       2.0.3                    pandas==2.0.3
16                 protobuf      4.25.3                 protobuf==4.25.3
17                pure25519       0.0.1                 pure25519==0.0.1
18                     py4j    0.10.9.7                   py4j==0.10.9.7
19                  pyarrow      15.0.2                  pyarrow==15.0.2
20                pycparser        2.22                  pycparser==2.22
21                    PyJWT       2.8.0                     PyJWT==2.8.0
22                pyOpenSSL      24.1.0                pyOpenSSL==24.1.0
23                  pyspark       3.4.1                   pyspark==3.4.1
24          python-dateutil 2.9.0.post0     python-dateutil==2.9.0.post0
25            python-dotenv       1.0.1             python-dotenv==1.0.1
26                     pytz      2024.1                     pytz==2024.1
27                   PyYAML       6.0.1                    PyYAML==6.0.1
28                 requests      2.31.0                 requests==2.31.0
29                      six      1.16.0                      six==1.16.0
30                     toml      0.10.2                     toml==0.10.2
31                   tzdata      2024.1                   tzdata==2024.1
32                  urllib3      1.26.6                  urllib3==1.26.6
33             vyper-config       1.2.1              vyper-config==1.2.1
34                 watchdog       4.0.0                  watchdog==4.0.

> sc <- spark_connect("test-spark-connect", method = "cde_connect")
> 
```
## Copy data to Spark and run simple dplyr test
```
> library(dplyr)
> iris_tbl <- sdf_copy_to(sc = sc, x = iris, overwrite = T)
> summaryStats <- iris_tbl %>%
   group_by(Species) %>%
   summarise(count = n(), 
             avgSepalLength = mean(Sepal_Length),
             avgSepalWidth= mean(Sepal_Width)) %>%
   collect
> summaryStats
# A tibble: 3 × 4
  Species    count avgSepalLength avgSepalWidth
  <chr>      <dbl>          <dbl>         <dbl>
1 versicolor    50           5.94          2.77
2 setosa        50           5.01          3.43
3 virginica     50           6.59          2.97

```
