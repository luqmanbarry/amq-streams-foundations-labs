## SETUP GRAFANA WITH PRELOADED KAFKA DASHBOARDS

1.  Create datasource.yaml provider file -- Reference grafana docs
			
2.  Create dashboard.yaml provider file -- Reference grafna docs
			
3. In dashboard.json file(s), make sure data source is set correctly -- replace ${DS_PROMETHEUS} by name value under datasources: of datasource.yaml file
			
4. Create config map: grafana-config for datasource.yaml and dashboard.yaml 

   * ```
     oc create configmap grafana-config --from-file=datasource.yaml=<path_to_datasource.yaml> --from-file=dashboard.yaml=<path_to_dashboard.yaml>
     ```
    *  Example:
        ```
        oc create cm grafana-config --from-file=datasource.yaml=./grafana-install/datasource.yaml --from-file=dashboard.yaml=./grafana-install/dashboard.yaml
        ```
			
5. Create config map to hold grafana dashboards
				
    ```
    oc create configmap grafana-config --from-file=<path_to_dashboards_dir>
    ```
    * Example:

        ```
        oc create configmap grafana-config --from-file=./kafka-dashboards
        ```
			
7. Mount the two config maps into grafana Deployment manifest -- 3 different mount paths using volumeMounts: and volumes:
    * /etc/grafana/provisioning/datasource.yaml for datasource provider config
    * /etc/grafana/dashboards/dashboard.yaml for dashboard provider config
    * /etc/grafana/dashboards/kafka for dashboards json contents
			
8. Lastly oc apply the grafana Deployment manifest

    ```
    oc apply -f grafana-install/ocp-grafana.yaml
    ```