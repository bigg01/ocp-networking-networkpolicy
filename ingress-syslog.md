# ingress-operator

https://docs.fluentbit.io/manual/input/syslog


```sh
            - name: SIX_ROUTER_SSL_VERIONS
              value: no-sslv3 no-tlsv10 no-tlsv11
            - name: TEMPLATE_FILE
              value: /var/lib/haproxy/conf/custom/haproxy-config.template
            - name: ROUTER_SYSLOG_ADDRESS
              value: '127.0.0.1:8514'
            - name: ROUTER_LOG_LEVEL
              value: info
            - name: ROUTER_SYSLOG_FORMAT
              value: >-
                client_ip=%ci\ client_port=%cp\ %ft\ %b/%s\ %Tq/%Tw/%Tc/%Tr/%Tt\
                status_code=%ST\ %B\ %CC\ %CS\ %tsc\ %ac/%fc/%bc/%sc/%rc\
                %sq/%bq\ %hr\ %hs\ %{+Q}r
            - name: SIX_TCP_ROUTER_SYSLOG_FORMAT
              value: >-
                client_ip=%ci\ client_port=%cp\ %ft\ %b/%s\ /%Tw/%Tc/%Tt\
                status_code=%ST\ %B\ %ac/%fc/%bc/%sc/%rc\ %sq/%bq\ %hr\ %hs
                
```
