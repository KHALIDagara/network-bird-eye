step 1 - run the grafana server with docker ( make sure docker is installed and working properly ) sudo docker run -d \
  --name=grafana \
  -p 3000:3000 \
  --restart=unless-stopped \
  grafana/grafana-oss:latest
step 2 - go to granfa settings .. add datasource .. select prometheus and scroll to bottom and add the following url : http://172.17.0.1:9090 (i tried localhost and 127.0.0.1 and they didn't work for some reason but this ip works  and i don't know why !! )
