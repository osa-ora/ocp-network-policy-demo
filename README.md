# OpenShift Network Policy Demo
---

In this demo, we will demostrate how you can deploy 2 tier application into OpenShift using 2 different projects/namespaces/tenant then test the application to identify the default behaviour (depending on OpenShift version), then deny all traffic, test it one more time, finally permit the traffic from that particular application to the DB (MySQL) and test it.

The demo is using the Quarkus sample application: https://github.com/osa-ora/Loyalty-Service

<img width="603" alt="Screenshot 2025-05-20 at 6 12 44 PM" src="https://github.com/user-attachments/assets/d95b3ad2-fb23-4a7d-94e2-0456a6be34ce" />


## Deploy the DB Tier

We will deploy it to the namespace "db-project" ..

```
oc new-project db-project
oc new-app mysql-persistent -p DATABASE_SERVICE_NAME=loyaltymysql -p  MYSQL_ROOT_PASSWORD=loyalty -p MYSQL_DATABASE=loyalty -p MYSQL_USER=loyalty -p MYSQL_PASSWORD=loyalty -p MEMORY_LIMIT=512Mi -p VOLUME_CAPACITY=512Mi -n db-project

oc patch dc loyaltymysql -n db-project --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/metadata/labels/app",
    "value": "loyaltymysql"
  }
]'
```

Once it is up and running, install the DB schema for our application:

```
//once the POD up and running ...
POD_NAME=$(oc get pods -l=name=loyaltymysql -o custom-columns=POD:.metadata.name --no-headers)

oc exec $POD_NAME -- mysql -u root loyalty -e "CREATE TABLE loyalty.loyalty_account (id INT NOT NULL AUTO_INCREMENT,balance INT NULL,tier INT NULL,enabled TINYINT NULL, PRIMARY KEY (id));CREATE TABLE loyalty.loyalty_transaction (id INT NOT NULL AUTO_INCREMENT,account_id INT NULL, points INT NULL, name VARCHAR(45) NULL, date DATETIME NULL, PRIMARY KEY (id));INSERT INTO loyalty.loyalty_account (id, balance, tier, enabled) VALUES ('1', '333', '1', '1');INSERT INTO loyalty.loyalty_account (id, balance, tier, enabled) VALUES ('2', '122', '1', '1');INSERT INTO loyalty.loyalty_account (id, balance, tier, enabled) VALUES ('3', '100', '2', '1');INSERT INTO loyalty.loyalty_transaction (id, account_id, points, name, date) VALUES ('1', '1', '100', 'KFC', '2019-01-19 14:55:02');INSERT INTO loyalty.loyalty_transaction (id, account_id, points, name, date) VALUES ('2', '1', '80', 'Carrefour', '2020-01-19 14:55:02');INSERT INTO loyalty.loyalty_transaction (id, account_id, points, name, date) VALUES ('3', '1', '75', 'Car Wash', '2020-02-19 14:55:02');INSERT INTO loyalty.loyalty_transaction (id, account_id, points, name, date) VALUES ('4', '2', '110', 'PizzaHut', '2020-01-19 14:55:02');"

```

## Deploy the Application Tier

We will deploy it to the namespace "app-project" ..

```
oc new-project app-project
oc new-app registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift~https://github.com/osa-ora/Loyalty-Service.git --name=quarkus-loyalty-java -n app-project
oc set env deployment/quarkus-loyalty-java DB_IP=loyaltymysql.db-project.svc.cluster.local
// for different port address
//oc set env deployment/quarkus-loyalty-java DB_PORT=3306

oc expose service/quarkus-loyalty-java -n app-project

oc patch deployment quarkus-loyalty-java -n app-project --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/metadata/labels/deployment",
    "value": "quarkus-loyalty-java"
  },
  {
    "op": "add",
    "path": "/spec/template/metadata/labels/app",
    "value": "quarkus-loyalty-java"
  }
]'
oc label namespace app-project name=app-project
```

First Test Case: The Default Behaviour 

```
//without any network policies ...
curl -s $(oc get route quarkus-loyalty-java -n app-project -o jsonpath='{.spec.host}')/loyalty/v1/balance/1

```

Second Test Case: The Deny All Policy Behaviour

```
//with deny all ..
oc apply -f https://raw.githubusercontent.com/osa-ora/ocp-network-policy-demo/refs/heads/main/policies/deny-all.yaml -n db-project
oc rollout restart deployment quarkus-loyalty-java -n app-project

curl -s $(oc get route quarkus-loyalty-java -n app-project -o jsonpath='{.spec.host}')/loyalty/v1/balance/1
```

Third Test Case: The Deny All Policy & Permit Application Specific Flow Behaviour

```
//with permit app to db only ..
oc apply -f https://raw.githubusercontent.com/osa-ora/ocp-network-policy-demo/refs/heads/main/policies/permit-policy.yaml -n db-project
oc rollout restart deployment quarkus-loyalty-java -n app-project

curl -s $(oc get route quarkus-loyalty-java -n app-project -o jsonpath='{.spec.host}')/loyalty/v1/balance/1

```


