# prom_email_notification_ec2_host_down_alertmanager
Prometheus  email notification when ec2 host down using Alertmanager

Steps
=====
1. Install node-exporter 
2. Install alertmanager
3. Install prometheus 
4. Install graphana

                [root@ip-172-31-16-13 PROM]# tree
                .
                |-- alertmanager
                |   |-- alertmanager
                |   |-- alertmanager.yml
                |   |-- amtool
                |   |-- data
                |   |   |-- nflog
                |   |   `-- silences
                |   |-- LICENSE
                |   `-- NOTICE
                |-- node-exporter
                |   |-- LICENSE
                |   |-- node_exporter
                |   `-- NOTICE
                `-- prometheus
                    |-- new_alert.rules.yml
                    `-- prometheus.yml

 
 1. Install node-exporter
 
 To extract the metrics of ec2/linux instance we need to istall node-exporter on that instance, it will fetch the metrics at /metrics endpoint.

    cd /opt/PROM
    wget https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz
    tar xvfz node_exporter-1.3.1.linux-amd64.tar.gz
    mv  node_exporter-1.3.1.linux-amd64 node_exporter
    cd node_exporter
    ./node_exporter &

Once node exporter is installed we can see metrics using : 

        curl http://127.0.0.1:9100/metrics

2. Install alert manager :

Download the alert manager from prometheus official website. For alertmanager to send email notification, need to do setup in the alertmanager.yml

      $ cat alertmanager.yml 
      global:
       resolve_timeout: 1m

      route:
       receiver: 'email-notifications'

      receivers:
      - name: 'email-notifications'
        email_configs:
        - to: tdashpute@gamil.com
          from: test@gmail.com
          smarthost: smtp.gmail.com:587
          auth_username: tdashpute@gmail.com
          auth_identity: tdashpute@gmail.com
          auth_password: PASSWORD
          send_resolved: true

      inhibit_rules:
        - source_match:
            severity: 'critical'
          target_match:
            severity: 'warning'
          equal: ['alertname', 'dev', 'instance']


    wget https://github.com/prometheus/alertmanager/releases/download/v0.23.0/alertmanager-0.23.0.linux-amd64.tar.gz
    tar xvfz alertmanager-0.23.0.linux-amd64.tar.gz
    mv alertmanager-0.23.0.linux-amd64.tar.gz alertmanager
    cd alertmanager
    ./alertmanager &

![image](https://user-images.githubusercontent.com/74225291/149608579-d4d25755-1d28-4454-a4d0-91c6a55632a2.png)


3. Install Prometheus :

Here we are setting up the prometheus on EC2 instance. We can setup it either using the promethues executable or using docker image.
We will follow the way of downloading executable and then will do the configuration as per requirement.

As we have already installed node-exporter and alert-manager we need to do intergrate it with prometheus.
Also to get the alerts we need to create rule file and it to the prometheus configration.

    # cat prometheus.yml 
    # my global config
    global:
      scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
      evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
      # scrape_timeout is set to the global default (10s).

    alerting:
      alertmanagers:
      - static_configs:
        - targets:
          - 172.31.16.13:9093

    # Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
    rule_files:
      - "new_alert.rules.yml"
      # - "second_rules.yml"

    # A scrape configuration containing exactly one endpoint to scrape:
    # Here it's Prometheus itself.
    scrape_configs:
      # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
      - job_name: 'prometheus'
        # metrics_path defaults to '/metrics'
        # scheme defaults to 'http'.
        static_configs:
        - targets: ['127.0.0.1:9090',172.31.16.13:9100]


a. Goto "https://github.com/prometheus" and search for prometheus.

        git clone https://github.com/prometheus/prometheus.git
        cd prometheus
        make build
        ./prometheus --config.file=promethues.yml
       
 Open the port 9090 to access the prometheus.
 http://"public_ip:9090"

Status of all the target's added.

![image](https://user-images.githubusercontent.com/74225291/149608908-d170dcb4-157b-4546-b776-2255ff075dd0.png)


We have configured 4 rules, all are inactive.
![image](https://user-images.githubusercontent.com/74225291/149608990-ad139f5d-e56e-43b9-b9cf-d4b6e4c3b1fa.png)

To trigger host down alert we can stop the node-exporter.

The alert will stay in pending state for 1 minutes then it will move to firing state.

![image](https://user-images.githubusercontent.com/74225291/149609172-146ae795-7c8d-4213-bebd-d9c245dda02c.png)

Now it is move to firing state.

![image](https://user-images.githubusercontent.com/74225291/149609211-c1f31096-3804-4ed5-bdc9-699d8a171f3d.png)


Now we can see it in alertmanager as well and we will get the mail notification as well for the same.

![image](https://user-images.githubusercontent.com/74225291/149609269-fdd8fb56-a1af-4a33-9198-83b47a0b453d.png)


4. Install graphana

If we want to see the visulatisation of all the metrcis we can install graphana as well. Once installation is done import 1860 chart.
Just for a change I have installed graphana using Docker.

    yum install docker -y
    service docker start

    docker run -d --name=grafana -p 3000:3000 grafana/grafana

    # docker ps
    CONTAINER ID   IMAGE             COMMAND                  CREATED          STATUS          PORTS                                       NAMES
    af91668f8deb   grafana/grafana   "/run.sh"                3 days ago       Up 3 days       0.0.0.0:3000->3000/tcp, :::3000->3000/tcp   grafana


Once grafana is up and running open "http://HOST_IP:3000"

Now goto configuration and add the prometheus host details to datasource.

![image](https://user-images.githubusercontent.com/74225291/149609749-02c217eb-6f01-4d17-9bef-6788bd59fdf4.png)

Import the chart 1860 which is for node-exporter.

![image](https://user-images.githubusercontent.com/74225291/149609810-6f804c85-9da8-4fe8-a88f-f0530e551527.png)

The final dashborard will look like this.

![image](https://user-images.githubusercontent.com/74225291/149608417-691f53fd-0d02-4bc6-a32d-36c4b6813ce0.png)




 
